# Resumen de la Prueba Técnica — Coupon Microservice

Este documento resume en forma práctica cómo se implementó y verificó la prueba técnica. Incluye decisiones de diseño, pasos ejecutados, problemas encontrados y cómo se resolvieron, así como instrucciones reproducibles para levantar y probar el sistema.

---

## Tabla de contenido

- [Versión A — FastAPI (Python)](#versión-a--fastapi-python)
  - [Resumen ejecutivo](#resumen-ejecutivo)
  - [Archivos y comandos](#archivos-y-comandos)
  - [Paso a paso de la implementación](#paso-a-paso-de-la-implementación)
  - [Problemas relevantes y decisiones](#problemas-relevantes-y-decisiones)
  - [Pruebas y resultados](#pruebas-y-resultados)
    - [Tabla de validación de endpoints](#tabla-de-validación-de-endpoints)
    - [E2E — Qué es y cómo ejecutarlo](#e2e--qué-es-y-cómo-ejecutarlo)
  - [Próximos pasos sugeridos](#próximos-pasos-sugeridos)
  - **Sección Técnica**
  - [Arquitectura y conceptos de diseño](#arquitectura-y-conceptos-de-diseño)
  - [Middleware y consumo seguro de API keys / tokens](#middleware-y-consumo-seguro-de-api-keys--tokens)
  - [FastAPI vs Laravel — Guía para entrevistas](#fastapi-vs-laravel--guía-para-entrevistas)
  - **Referencia**
  - [Glosario técnico](#glosario-técnico)

- [Versión B — Laravel (PHP)](#versión-b--laravel-php)
  - [Resumen ejecutivo](#resumen-ejecutivo-1)
  - [Archivos y comandos](#archivos-y-comandos-1)
  - [Paso a paso de la implementación](#paso-a-paso-de-la-implementación-1)
  - [Problemas relevantes y decisiones](#problemas-relevantes-y-decisiones-1)
  - [Pruebas y resultados](#pruebas-y-resultados-1)
    - [Tabla de validación de endpoints](#tabla-de-validación-de-endpoints-1)
  - [Próximos pasos sugeridos](#próximos-pasos-sugeridos-1)
  - **Sección Técnica**
  - [Arquitectura y conceptos de diseño](#arquitectura-y-conceptos-de-diseño)
  - [Middleware y consumo seguro de API keys / tokens](#middleware-y-consumo-seguro-de-api-keys--tokens)
  - [FastAPI vs Laravel — Guía para entrevistas](#fastapi-vs-laravel--guía-para-entrevistas)
  - **Referencia**
  - [Glosario técnico](#glosario-técnico)

---

# Versión A — FastAPI (Python)

---

## Resumen ejecutivo

- Implementación del microservicio de cupones en Python usando FastAPI.
- Servicios auxiliares: WordPress + WooCommerce (docker), MySQL (docker) para el entorno de integración.
- Base de datos: SQLite local (`version_a.db`), independiente del MySQL usado por WordPress.
- Tests unitarios: 5 pruebas con `pytest` (cobertura de reglas de negocio básicas).
- Documentación: OpenAPI (`openapi.json`) y colección Postman generada.

---

## Archivos y comandos

### Versión A — FastAPI (Python)

- Implementación: `version-a/` — microservicio en Python usando FastAPI, `SQLModel` y `pydantic`.
- Archivos clave:
  - `version-a/app/` — routes, `models.py`, `schemas.py`, `services/`
  - `version-a/tests/` — `test_basic.py` (5 tests unitarios)
  - `version-a/scripts/` — `setup-woocommerce.sh`, `entrypoint.sh`, `test_services.py`, `generate_postman_collection.py`, `export_openapi.py`
  - `version-a/postman_collection.json` — colección Postman generada
  - `version-a/openapi.json` — OpenAPI exportado desde la app

Comandos principales:

```bash
cd version-a
docker compose up --build -d              # levantar stack completo
source .venv311/bin/activate             # activar entorno local
pytest -v                                # ejecutar tests unitarios
PYTHONPATH=. python3 scripts/export_openapi.py
python3 scripts/generate_postman_collection.py
```

Acceso a documentación:
- Swagger UI: `http://localhost:8000/docs` (o `/swagger` redirige a `/docs`)
- Redoc: `http://localhost:8000/redoc`

---

## Paso a paso de la implementación

1. **Scaffold inicial**
   - Crear estructura `version-a/` con `app`, `tests`, `scripts`, `Dockerfile` y `docker-compose.yml`.
   - Definir dependencias en `requirements.txt` (`fastapi`, `uvicorn`, `sqlmodel`, `pydantic` v1, `pytest`, etc.).

2. **Implementación del microservicio**
   - `app/models.py`: modelos SQLModel para `Coupon`, `CouponUsage`, `CouponEmailRestriction`.
   - `app/schemas.py`: Pydantic schemas para validación de requests/responses.
   - `app/services/woo.py`: `FakeWooCommerceService` que simula resolución de categorías/productos y creación de cupones externos.
   - `app/main.py`: endpoints REST bajo `/api/v1` (health, create, bulk, list, get, update, delete, validate, apply).

3. **Reglas de negocio implementadas**
   - Birthday: forzar 15%.
   - Referral: forzar $3.000.
   - Night sale: máximo 50%.
   - Campaign: requiere `expires_at`.
   - Email restrictions y `usage_count` se registran y validan localmente.

4. **Tests unitarios**
   - Archivo: `version-a/tests/test_basic.py` (5 tests).
   - Ejecutados localmente con Python 3.11 en virtualenv (`.venv311`): `pytest -v` → 5 passed.

5. **Docker + WordPress/WooCommerce**
   - `docker-compose.yml` levanta: `db` (MySQL), `wordpress`, `wc-setup` (WP-CLI provisioning) y `app` (FastAPI).
   - Problema: `wc-setup` fallaba con "Error establishing a database connection" al no recibir variables `WORDPRESS_DB_*`. Solución: pasar las variables de entorno de DB al contenedor WP-CLI.
   - Generación de claves WC: se añadieron scripts PHP (`tmp_create_keys.php`) y se usó `wp eval-file` para insertar claves en `wp_woocommerce_api_keys` y escribir `version-a/shared/wc.env`.

6. **Verificación E2E**
   - Se creó `version-a/scripts/test_services.py` que valida la API FastAPI y llama a endpoints de WC cuando es posible.
   - Flujo completo: health → creación de cupones (birthday/referral/night_sale/campaign) → validación → apply.

7. **Documentación y artefactos**
   - OpenAPI exportado a `version-a/openapi.json`.
   - Swagger UI disponible en `/docs`; `/swagger` redirige ahí por conveniencia.
   - Postman: `version-a/postman_collection.json`.

---

## Problemas relevantes y decisiones

- **Pydantic / SQLModel**: se fijó `pydantic==1.10.11` para compatibilidad con `sqlmodel==0.0.8`.
- **Virtualenvs**: se creó `.venv311` (Python 3.11) porque Python 3.14 produjo incompatibilidades con algunas dependencias.
- **Diseño**: se priorizó código sencillo y claridad sobre exhaustividad; el `FakeWooCommerceService` es deliberado para poder ejecutar tests sin infraestructura externa.
- **WP-CLI provisioning**: comandos `wp wc create` no disponibles en todas las instalaciones; se creó fallback PHP para insertar claves directamente en la DB.

---

## Pruebas y resultados

Tests unitarios: **5 passed** (`pytest -v` con Python 3.11).

### Tabla de validación de endpoints

| Endpoint | Test | Resultado |
|---|---|---|
| GET /api/v1/health | Health check | ok: true, wc: true ✓ |
| POST /api/v1/coupons | Birthday (50→15) | amount: 15.0 ✓ |
| POST /api/v1/coupons | Referral (5000→3000) | amount: 3000.0 ✓ |
| POST /api/v1/coupons | Night sale (80→max 50) | amount: 50.0 ✓ |
| POST /api/v1/coupons | Campaign sin expires_at | 422 ✓ |
| POST /api/v1/coupons | Campaign con expires_at | Creado ✓ |
| POST .../validate | Email no autorizado | valid: false ✓ |
| POST .../apply | Aplicar cupón | usage_count: 1 ✓ |
| GET /api/v1/coupons | Listar cupones | 4 cupones ✓ |

### E2E — Qué es y cómo ejecutarlo

E2E (End-to-End) validan flujos completos: cliente → API → persistencia → servicios externos. Detectan errores de integración que no salen en tests unitarios.

```bash
# Con Docker stack completo
cd version-a
docker compose up --build -d
PYTHONPATH=. python3 scripts/test_services.py

# Local (sin Docker WordPress)
cd version-a
source .venv311/bin/activate
PYTHONPATH=. python3 scripts/test_services.py
```



## Próximos pasos sugeridos

1. Reemplazar `FakeWooCommerceService` por un cliente real `httpx` que use `shared/wc.env`.
2. Persistir la DB del microservicio en un volumen Docker entre reinicios.
3. Añadir tests de integración que cubran sincronización real con WooCommerce.
4. Configurar pipeline CI/CD con linters, tests y build de imagen.

---
---

# Versión B — Laravel (PHP)

---

## Resumen ejecutivo

- Implementación del microservicio de cupones en PHP usando Laravel.
- Integración directa con WooCommerce a través de `WooCommerceService`.
- Base de datos: MySQL (compartida con el entorno Docker).
- Autenticación por middleware `ApiKeyAuth` con header `X-API-Key`.
- Migraciones propias: `coupons`, `coupon_email_restrictions`, `coupon_usages`.

---

## Archivos y comandos

- Implementación: `version-b/` — aplicación Laravel con integración WooCommerce.
- Archivos clave:
  - `version-b/app/Http/Controllers/` — controladores REST
  - `version-b/app/Http/Middleware/` — autenticación por `X-API-Key`
  - `version-b/app/Services/CouponService.php` + `WooCommerceService.php`
  - `version-b/app/Models/` — `Coupon`, `CouponUsage`, `CouponEmailRestriction`, `User`
  - `version-b/database/migrations/` — migraciones de cupones, restricciones y usos
  - `version-b/config/woocommerce.php` — configuración WC

Comandos principales:

```bash
cd version-b
composer install
cp .env.example .env && php artisan key:generate
php artisan migrate
php artisan serve --host=0.0.0.0 --port=8001
```

---

## Paso a paso de la implementación

1. **Scaffold inicial** — estructura Laravel estándar + `composer.json` con dependencias.
2. **Modelos y migraciones** — `Coupon`, `CouponUsage`, `CouponEmailRestriction` con Eloquent ORM.
3. **Reglas de negocio** — implementadas en `CouponService`: birthday 15%, referral $3.000, night sale máx 50%, campaign requiere `expires_at`.
4. **Integración WooCommerce** — `WooCommerceService` con contrato (`WooCommerceServiceInterface`) para facilitar testing.
5. **Middleware** — `ApiKeyAuth` valida `X-API-Key` en cada request.
6. **Routes** — endpoints REST bajo `/api/v1` en `routes/api.php`.
7. **Tests** — `tests/Feature/CouponApiTest.php` con `FakeWooCommerceService`.

---

## Problemas relevantes y decisiones

- **Contrato de servicio**: `WooCommerceServiceInterface` separa contrato de implementación, permitiendo inyectar el fake en tests.
- **Autenticación**: middleware simple `X-API-Key`; en producción se recomienda Laravel Sanctum o Passport.
- **Compatibilidad Docker**: `docker-compose.yml` incluido para levantar MySQL + app Laravel.

---

## Pruebas y resultados

### Tabla de validación de endpoints

| Endpoint | Test | Resultado |
|---|---|---|
| GET /api/v1/health | Health check | ok: true ✓ |
| POST /api/v1/coupons | Birthday (50→15) | amount: 15.0 ✓ |
| POST /api/v1/coupons | Referral (5000→3000) | amount: 3000.0 ✓ |
| POST /api/v1/coupons | Night sale (80→max 50) | amount: 50.0 ✓ |
| POST /api/v1/coupons | Campaign sin expires_at | 422 ✓ |
| POST .../validate | Email no autorizado | valid: false ✓ |

---

## Próximos pasos sugeridos

1. Completar cobertura de tests de integración con WooCommerce real.
2. Añadir autenticación con Laravel Sanctum para mayor seguridad.
3. Configurar pipeline CI/CD (GitHub Actions + PHPUnit).
4. Documentación OpenAPI con L5-Swagger o similar.

---
---

# Sección Técnica

> Compartida entre Versión A y Versión B. Agrupa arquitectura, patrones de seguridad y guía comparativa para entrevistas.

---

## Arquitectura y conceptos de diseño

- **Arquitectura por capas**: la aplicación sigue una separación clara entre Routes (API layer), Services (integración y lógica externa) y Models/DB (persistence layer). Esto facilita pruebas unitarias y la sustitución de componentes (por ejemplo, reemplazar `FakeWooCommerceService` por un cliente HTTP real).

- **Bounded Context y responsabilidad**: el microservicio es el _source of truth_ para reglas de negocio específicas de cupones (validaciones, restricciones, tracking de usos). WooCommerce se trata como un sistema externo que persiste cupones para el front-end/tienda.

- **Diseño de API**: RESTful, versionado en la ruta (`/api/v1`) y uso de esquemas Pydantic para validación estricta de payloads. Respuestas estandarizadas y modelos de salida (`CouponOut`) para facilitar consumidores.

- **Modelado de datos**: uso de `SQLModel` (ORM ligero sobre SQLAlchemy) para modelos tipados y migraciones simples. Se usa SQLite para facilidad en la prueba técnica; en producción se recomendaría una base transaccional gestionada (MySQL/Postgres).

- **Transacciones y consistencia**: operaciones que modifican estado local se envuelven en transacciones SQLModel; la sincronización con WooCommerce es eventual (simulada). Para integraciones reales, diseñar compensaciones o colas (retry/backoff) para garantizar idempotencia y consistencia eventual.

- **Integración externa**: las llamadas a WooCommerce deben manejar:
	- Autenticación con claves (`consumer_key`/`secret`) guardadas de forma segura.
	- Retries y circuit-breaker para tolerancia a fallos.
	- Idempotencia en creación/actualización de recursos externos.

- **Testing strategy**:
	- Unit tests: validar lógica de reglas sin infra externa (se usan `FakeWooCommerceService`).
	- Integration tests: levantar stack Docker (`wordpress` + `db`) para verificar flujos E2E.
	- E2E: scripts reproducibles (`scripts/test_services.py`) para validar comportamiento con la pila completa.

- **Observabilidad**:
	- Logging: mensajes estructurados en puntos clave (creación, validación, errores externos).
	- Metrics: exponer contadores básicos (requests, errores, latencias) vía Prometheus o similar.
	- Tracing: añadir trazas distribuidas para correlacionar llamadas con WooCommerce si se integra en producción.

- **Seguridad**:
	- Autenticación simple por `X-API-Key` en la prueba; en producción usar OAuth2/JWT o mTLS según responsabilidades.
	- No almacenar secretos en el repo: `shared/wc.env` es para demo; usar secretos de entorno/secret manager en producción.
	- Validación estricta de inputs (Pydantic) para mitigar inyección/errores.

- **Despliegue y operaciones**:
	- Contenerizar con `Dockerfile` y orquestación simple con `docker-compose` para entorno de pruebas.
	- Para producción, empaquetar en imágenes inmutables y desplegar en Kubernetes o servicios gestionados, con readiness/liveness probes, volúmenes para persistencia y backups para la DB.

- **CI/CD**:
	- Pipeline debe ejecutar linters, tests unitarios, build de imagen y tests de integración en entorno aislado.
	- Publicar imágenes a registry y despliegue controlado con rollbacks.

## Cómo explicar FastAPI y Laravel (para entrevistas)

Esta sección resume puntos claros y directos que puedes usar en una entrevista para explicar cómo funcionan FastAPI (Python) y Laravel (PHP) y sus diferencias/ventajas.

FastAPI (Python)
- Propósito: framework moderno ASGI para construir APIs HTTP rápidas y asíncronas con tipado y validación automática.
- Punto fuerte: integración estrecha con type hints de Python y Pydantic — genera validación de entrada/salida y documentación OpenAPI automáticamente.
- Request lifecycle (resumen):
	1. Uvicorn/ASGI recibe la petición.
	2. FastAPI resuelve la ruta y los parámetros usando introspección de type hints.
	3. Pydantic valida y parsea el body/params a modelos Python.
	4. Dependencias (Depends) se resuelven (inyección de dependencias ligera).
	5. Se ejecuta la función del endpoint (puede ser async o sync). Si es async, el event loop lo administra.
	6. Resultado se serializa a JSON usando Pydantic y se devuelve; OpenAPI y Swagger se exponen automáticamente.
- Características clave:
	- Soporta `async`/`await` nativo (alto rendimiento para I/O-bound).
	- `Depends` para DI simple y reutilizable (autenticación, sessions, etc.).
	- Genera `openapi.json`, `docs` y `redoc` automáticamente.
	- Muy fácil de testear con `TestClient` (Starlette/requests compatible).
- Uso en backend: servicios HTTP, microservicios, gateways y APIs con alta concurrencia.

Laravel (PHP)
- Propósito: framework web completo para aplicaciones PHP, con enfoque en productividad y convenciones (MVC).
- Punto fuerte: gran ecosistema (Eloquent ORM, queues, jobs, events, blade templates) y curvas de adopción suaves para aplicaciones monolíticas y APIs.
- Request lifecycle (resumen):
	1. Servidor HTTP (Nginx/Apache + PHP-FPM) recibe la petición.
	2. `public/index.php` inicia el kernel de Laravel.
	3. Middleware global y de ruta se ejecutan (autenticación, CORS, etc.).
	4. Router resuelve controlador/acción.
	5. Controller ejecuta lógica, usa servicios/repositories, y devuelve `Response`.
	6. Eloquent ORM maneja persistencia; eventos y jobs pueden dispararse para trabajo asíncrono.
- Características clave:

## Glosario técnico

- E2E (End-to-End): Pruebas que validan flujos completos del sistema, desde la entrada del usuario hasta la persistencia y sistemas externos.
- API (Application Programming Interface): Conjunto de reglas y contratos que permiten la interacción entre sistemas mediante llamadas HTTP u otros protocolos.
- JWT (JSON Web Token): Token firmado que contiene claims; se usa para autenticación/authorization sin consultar DB en cada petición.
- HMAC (Hash-based Message Authentication Code): Método para verificar integridad/autenticidad de mensajes usando una clave secreta.
- Hashing (bcrypt, Argon2): Transformación unidireccional usada para almacenar verificadores de secretos sin poder recuperar el valor original.
- TLS/HTTPS: Protocolo de seguridad para cifrado en tránsito; obligatorio en producción.
- OAuth2: Protocolo de autorización que permite emitir tokens de acceso y refresh tokens para clientes.
- DI (Dependency Injection): Patrón para inyectar dependencias en clases/funciones (FastAPI `Depends`, Service Container en Laravel).
- ORM (Object-Relational Mapping): Biblioteca que mapea objetos del lenguaje a registros de la base de datos (Eloquent, SQLModel/SQLAlchemy).
- Idempotencia: Propiedad de una operación que puede ejecutarse múltiples veces sin cambiar el resultado más allá de la primera ejecución.
- Circuit breaker: Patrón para desactivar temporalmente llamadas a un servicio externo fallido y evitar saturarlo.
- Rate limiting: Límite de peticiones por unidad de tiempo para proteger la API contra abuso.
- CI/CD: Integración continua y despliegue continuo; pipelines que ejecutan tests, linters y despliegan artefactos.
- Secret Manager: Servicio para almacenar secretos (ej. Vault, AWS Secrets Manager) de forma segura fuera del repo.
- Soft delete: Técnica que marca registros como borrados sin eliminarlos físicamente (`deleted_at`).
- OpenAPI / Swagger: Especificación que describe APIs REST y genera documentación interactiva (`/docs`, `/redoc`).
	- Eloquent ORM para Active Record (relaciones, scopes, eager loading).
	- Sistema de Service Container (DI) potente y bindings.
	- Jobs/queues, eventos y broadcasting integrados.
	- Blade para templates y gran ecosistema (Horizon, Passport, Sanctum).
- Uso en backend: aplicaciones monolíticas, APIs REST, backend para tiendas, CRMs.

Comparativa rápida (entrevista):
- Concurrency: FastAPI (ASGI + async) es superior para I/O-bound y alta concurrencia; Laravel tradicionalmente usa procesos PHP-FPM (concurrency por worker), aunque puede integrarse con Swoole para async.
- Tipado: FastAPI se apoya en type hints y Pydantic; Laravel es dinámico pero ofrece PHP type hints y herramientas estáticas (PHPStan, Psalm).
- Ecosistema: Laravel tiene más funcionalidades integradas (ORM, queues, autenticación) listas para usar; FastAPI suele combinarse con librerías específicas (SQLModel/SQLAlchemy, Celery, etc.).
- Productividad vs Performance: Laravel acelera desarrollo con convenciones; FastAPI suele dar mejor rendimiento raw para APIs.

Puntos de conversación técnico para entrevistas
- Explica el ciclo de una petición y dónde puedes colocar autenticación, caching y tracing.
- Habla de validación y modelado (Pydantic vs Eloquent/DTOs).
- Menciona testing: unit + integration (TestClient en FastAPI; PHPUnit/Laravel HTTP tests en Laravel).
- Menciona despliegue: ASGI + Uvicorn/Gunicorn + container vs PHP-FPM + Nginx + container/K8s.
- Seguridad: validación, manejo de secretos, protección CSRF (Laravel cuando hay forms), rate limiting y autenticación (JWT/OAuth2).

Ejemplo de respuesta corta para entrevista
"FastAPI es un framework ASGI que usa type hints y Pydantic para validar requests automáticamente y exponer OpenAPI; su modelo async lo hace ideal para microservicios de alta concurrencia. Laravel es un framework MVC muy completo con Eloquent ORM y un potente service container, ideal para aplicaciones monolíticas que requieren rapidez de desarrollo y un ecosistema rico."




## Detalle técnico: clases, middleware y consumo seguro de API keys / tokens

Esta sección describe a bajo nivel las clases y el flujo para autenticar y consumir la API, y recomienda patrones seguros para tokens cifrados o rotación de claves.

1) Clases y módulos principales
- FastAPI (`version-a`):
	- `version-a/app/main.py` → `require_api_key(x_api_key: str = Header(...))` (dependencia usada como middleware lógico).
	- `version-a/app/services/woo.py` → `FakeWooCommerceService` (implementación de ejemplo/contract).
	- `version-a/app/models.py` → `Coupon`, `CouponUsage`, `CouponEmailRestriction` (modelos SQLModel).

- Laravel (`version-b`):
	- `App\Http\Middleware\ApiKeyAuth` → middleware con método `handle($request, Closure $next)` que valida `X-API-Key`.
	- `App\Services\CouponService` → orquesta creación/validación/registro de usos.
	- `App\Services\WooCommerceServiceInterface` + `WooCommerceService` → contrato + implementación para llamadas WC.

2) Flujo de autenticación (resumen)
- Cliente envía header `X-API-Key: <token>` en cada petición.
- Middleware (Laravel) o dependencia (FastAPI) extrae el header y verifica la validez:
	- Comparación directa (solo para demos): comparar con valor en `ENV`.
	- Producción recomendado: almacenar solo el hash del token en el servidor y comparar con `Hash::check()` (Laravel) o `bcrypt.checkpw()` (Python).
- Si la verificación falla → `401 Unauthorized`; si pasa, petición continúa y se inyecta contexto necesario.

3) Consumo de tokens cifrados / patrones seguros
- Almacenamiento de secretos: nunca en el repo. Usar `ENV`, `Vault`, AWS Secrets Manager o GitHub Secrets.
- Hashing vs cifrado:
	- Hash (unidireccional, p.ej. bcrypt): ideal si solo necesitas verificar la clave (no recuperar el valor original).
	- Cifrado simétrico (p.ej. AES) o asimétrico (RSA) cuando necesitas recuperar un secreto; almacenar la clave de cifrado en un secret manager.
- Tokens firmados (JWT): recomendable para short-lived tokens; validar firma (`RS256`) y claims (`exp`, `iss`, `aud`) en middleware.
- Rotación y revocación: usar tokens de corta vida y refresh tokens, o mantener una lista de revocación en DB/cache para invalidación inmediata.

4) Ejemplo conceptual de verificación (pseudo-code)
- FastAPI (concepto):

	def require_api_key(x_api_key: str = Header(...)):
			stored_hash = load_from_env_or_secret_manager()
			if not bcrypt.checkpw(x_api_key.encode(), stored_hash):
					raise HTTPException(401)

- Laravel (concepto en `ApiKeyAuth`):

	public function handle($request, Closure $next) {
			$key = $request->header('X-API-Key');
			$storedHash = config('services.api.key_hash');
			if (!Hash::check($key, $storedHash)) {
					return response()->json(['message' => 'Unauthorized'], 401);
			}
			return $next($request);
	}

5) TLS y cifrado en reposo
- Siempre usar HTTPS/TLS en producción. Localmente `docker compose` es para pruebas; no usarlo en producción sin TLS.
- Para secretos en disco, cifrar o usar control de acceso estricto; preferir secret managers.

6) Recomendaciones operacionales
- Emite claves con fecha de expiración y soporte rotación automatizada.
- Monitoriza intentos de fallo de autenticación y aplica rate limiting por IP/key.
- Para integraciones externas (WC consumer keys) almacenar en `shared/wc.env` solo para demo; en producción usar secrets y permisos estrictos.

Si quieres que adapte este documento (más detalles, comandos específicos, capturas de logs o links a commits), dime qué parte quieres ampliar y lo hago.

