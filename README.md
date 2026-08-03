# Sistema Integral del Comité de Agua Potable y Alcantarillado de Belisario Domínguez

Diseño arquitectónico completo para un **ERP especializado en organismos operadores de agua potable** en comunidades rurales, donde la mayoría de los domicilios no tienen dirección formal (se ubican por lote, manzana, domicilio conocido, referencias físicas, croquis o coordenadas GPS).

## Documentos de diseño

| Documento | Contenido |
|---|---|
| [01-arquitectura.md](docs/01-arquitectura.md) | Arquitectura general, dominios/módulos, stack tecnológico, diagrama de componentes |
| [02-datos.md](docs/02-datos.md) | Modelo entidad-relación, base de datos normalizada, catálogo de tablas, relaciones |
| [03-casos-de-uso-flujos.md](docs/03-casos-de-uso-flujos.md) | Casos de uso, flujos operativos, flujo financiero completo |
| [04-api.md](docs/04-api.md) | Diseño de APIs REST, convenciones, ejemplos |
| [05-permisos-auditoria-seguridad.md](docs/05-permisos-auditoria-seguridad.md) | Modelo RBAC, estrategia de auditoría, seguridad y privacidad |
| [06-georreferenciacion.md](docs/06-georreferenciacion.md) | Estrategia de georreferenciación para expedientes rurales |
| [07-modulos-operativos.md](docs/07-modulos-operativos.md) | Lecturas de medidores, órdenes de trabajo, infraestructura georreferenciada |
| [08-escalabilidad-respaldo.md](docs/08-escalabilidad-respaldo.md) | Escalabilidad, respaldos, recuperación ante desastres |
| [09-reglas-de-negocio.md](docs/09-reglas-de-negocio.md) | Business Rules Catalog (BR-xxx) con trazabilidad regla → función SQL → operación OpenAPI |

## Fases de implementación

El diseño es la **especificación**; el código es una implementación mecánica de la misma. Orden estricto para que el código no contamine el diseño:

| Fase | Entregable | Estado |
|---|---|---|
| 1 | **Base de datos** — SQL de producción numerado bajo `schema/` (PostgreSQL + PostGIS: tablas, PK/FK, CHECK, UNIQUE, índices, secuencias, triggers, vistas, funciones, soft delete, auditoría). Debe quedar funcional sin una línea de backend. **26 migraciones aplicadas y validadas en PostgreSQL real (PGlite / PG16 WASM): 29/29 aserciones OK del harness E2E** (ver [Pruebas del esquema](#pruebas-del-esquema)). | Listo |
| 2 | **OpenAPI 3.1** — contrato oficial modular `openapi/` (~120 rutas, 157 operaciones): schemas, DTOs, HTTP, errores ProblemDetails (RFC 9457), JWT, paginación/filtros, versionado `/api/v1`, idempotencia (`Idempotency-Key`) y concurrencia optimista (`ETag`/`If-Match`) en operaciones financieras y recursos editables, con transiciones de estado como acciones explícitas. Fuente en `openapi/src/`, consolidado `openapi/openapi.yaml` generado y validado (ver [Contrato OpenAPI](#contrato-openapi)). | Listo |
| 3 | **Backend** — capa delgada FastAPI sobre los motores SQL: DTO, JWT + RBAC, transacción por petición, gateway por dominio que invoca las funciones SQL oficiales y traduce errores a ProblemDetails. Slice vertical Caja (cobrar) + auth funcionando con pruebas sin BD (ver [Backend](#backend-fase-3)). | En curso |
| 4 | **Frontend** — pantallas administrativas, dashboard, mapa, caja, cobranza, inventario, infraestructura. | Pendiente |
| 5 | **Móvil** — offline-first, GPS, fotografía, firma, lecturas, órdenes de trabajo, sincronización. | Pendiente |

### Decisiones añadidas antes de la Fase 1

1. **Motor de Facturación (Billing Engine)** — dominio propio (`bil.*`): ciclos de facturación, generación automática de **cargos** por servicio (tarifa fija + consumo + cuota de servicio), y conversión en **adeudos**. Es **independiente de Caja**: Caja solo cobra lo que el Billing Engine generó.
2. **Motor de Tarifas** — tarifas versionadas con `vigencia_inicio`/`vigencia_fin` y tipología (`DOMESTICA, COMERCIAL, INDUSTRIAL, SOCIAL, ADULTO_MAYOR, TEMPORAL, POR_CONSUMO, FIJA`). Los datos históricos jamás se modifican; cada periodo factura con la tarifa vigente a su corte.
3. **Jerarquía física: Predio → Servicio → Titular** — la entidad raíz es el **Predio** (ubicación física de la toma), no la persona. El titular es una **relación histórica** del predio (`predio_titulares` con vigencias). Al cambiar de propietario (venta/herencia) no se migran datos ni se rompe el historial operativo y financiero de la toma.

## Resumen ejecutivo

- **Propósito:** administrar por completo la operación del comité: expedientes georreferenciados, servicios, cobranza, pagos, adeudos, multas, donaciones, egresos, inventario, lecturas, órdenes de trabajo, infraestructura, reportes y auditoría total.
- **Enfoque:** arquitectura modular orientada a dominios (DDD), monólito modular como punto de partida con posibilidad de evolución a servicios; base de datos normalizada (3FN) con geolocalización nativa (PostGIS).
- **Dato central:** el **predio georreferenciado** (ubicación física de la toma) como entidad raíz; el **servicio** (agua/drenaje) como unidad facturable; el **titular** como relación histórica del predio. No se exige dirección formal; la ubicación es un modelo propio de referencias + punto GPS.
- **Motores de negocio:** **Motor de Facturación** (genera cargos/adeudos por ciclo, independiente de caja) y **Motor de Tarifas** (tarifas versionadas con vigencia y tipología), ambos en la base de datos (funciones SQL) desde la Fase 1.
- **Multiusuario:** control de acceso por roles (RBAC) y bitácora de auditoría inmutable de toda acción.
- **Ruralidad:** modo **offline-first** para captura en campo (lecturas, órdenes de trabajo, GPS) con sincronización posterior.
- **Tecnología base recomendada:** PostgreSQL + PostGIS, API REST, frontend web (PWA) + captura móvil, almacenamiento de archivos (fotografías, croquis, facturas).

## Decisiones clave de diseño

1. **Una sola base de datos relacional normalizada** con esquemas por dominio; no se utilizan bases separadas por módulo.
2. **Entidad raíz física (Predio).** `exp.predios` es la ubicación física; `ser.servicios` (la toma) cuelga del predio y es la unidad facturable; el titular es historial en `exp.predio_titulares`. El expediente administrativo (documentos, fotos, croquis) se adjunta al predio.
3. **Modelo de ubicación no-direccional:** punto GPS + jerarquía (localidad → colonia → lote/manzana → domicilio conocido → referencias) + croquis; el mapa es herramienta de primer nivel.
3. **Finanzas de doble partida simplificada:** todo movimiento es inmutable; se generan folios secuenciales por módulo y tipo; balance = ingresos − egresos con cierre de caja diario.
4. **Adeudos auto-calculados** a partir de tarifas, ciclos de facturación, recargos e intereses configurables; condonaciones, congelamientos y convenios quedan historizados.
5. **Auditoría append-only:** los registros nunca se eliminan físicamente; borrado lógico + bitácora con valores anteriores/nuevos en JSON.
6. **Georreferenciación como servicio transversal:** expedientes, tomas, medidores, infraestructura y órdenes de trabajo comparten el mismo modelo geoespacial.
7. **Extensible:** los módulos operativos (lecturas, órdenes de trabajo, infraestructura) se integran con cobranza y finanzas, convirtiendo al sistema en herramienta de operación, no solo de caja.

## Pruebas del esquema

Las migraciones de Fase 1 se validan con un **harness reproducible** en
[`tests/schema/`](tests/schema/README.md) contra PostgreSQL real (PGlite, PG16
WASM, sin servidor). Es un activo permanente del proyecto, no una utilidad
temporal: detecta errores que el análisis estático no ve (ya encontró dos bugs
reales en el trigger de auditoría y en el cálculo de tarifas por tramos).

```bash
cd tests/schema
npm install
npm test        # → Resultado: 29 OK, 0 FAIL (exit 0)
```

**Qué valida:** aplicación de las 26 migraciones, compilación de todo el
PL/pgSQL, motor de tarifas, recargos/intereses y el flujo E2E completo
(expediente → servicio → lectura → facturación → caja → anulación → auditoría
hash chain → cierre de turno).

**Qué no valida:** PostGIS real (geometry/GiST/ST_*) y pgcrypto real se
sustituyen por stubs; tampoco RLS, concurrencia ni rendimiento. El workflow de
CI [`.github/workflows/schema.yml`](.github/workflows/schema.yml) cierra esa
brecha con un segundo trabajo que aplica las migraciones en PostgreSQL +
PostGIS + pgcrypto reales.

Pipeline de integración (cualquier cambio en `schema/*.sql` que rompa la base
detiene la entrega):

```
Migraciones SQL → crear BD limpia → aplicar migraciones → smoke.js → pasar/fallar
```

## Contrato OpenAPI

El contrato API (Fase 2) vive en [`openapi/`](openapi/README.md). Es **fuente
modular** (`openapi/src/`) de la que se genera el consolidado
`openapi/openapi.yaml`, y es la especificación de entrada del backend (OpenAPI →
Backend, no al revés).

```bash
cd openapi
npm test          # build (consolidado) + validación OpenAPI 3.1
```

**Qué incorpora** (más allá de schemas/DTOs/HTTP):

- **Errores** estandarizados como ProblemDetails (RFC 9457) con código de negocio estable.
- **Idempotencia:** `Idempotency-Key` obligatoria en operaciones financieras (cobrar, pagos,
  anular recibo, donaciones, egresos, multas, convenios, facturación).
- **Concurrencia optimista:** `ETag` + `If-Match` (`412`) en expedientes, servicios, tarifas,
  OT e infraestructura.
- **Transiciones de estado explícitas:** `POST /ordenes-trabajo/{id}/assign|start|complete|validar|cerrar|cancel`
  y `POST /servicios/{id}/cambio-estado`; el campo `estado` es de solo lectura.
- **Geometrías GeoJSON** (RFC 7946), paginación `{ data, pagination }`, filtros, `q`, `sort/order`.

La CI ([`.github/workflows/openapi.yml`](.github/workflows/openapi.yml)) valida la fuente,
regenera el consolidado y verifica que esté sincronizado con el commit.

## Backend (Fase 3)

El backend vive en [`backend/`](backend/README.md): FastAPI sobre los **motores
SQL** de la Fase 1 (PostgreSQL = dominio, FastAPI = orquestador, no se duplica
lógica de negocio en Python). Arquitectura por dominios
(`src/domains/<dominio>/{dto,gateway,service,routes}.py`) con un gateway por
dominio como única vía de acceso a SQL.

```bash
cd backend
.venv/bin/python -m pytest   # 24 pruebas sin BD (DTO, auth, invocación, errores)
.venv/bin/ruff check .
```

Las reglas de negocio son trazables en
[docs/09-reglas-de-negocio.md](docs/09-reglas-de-negocio.md) (BR-xxx → función
SQL → operación OpenAPI → servicio).

## Cómo usar este diseño

Los documentos son la **especificación de entrada** para que otra IA (o un equipo) genere el código: contienen el contrato de datos, las reglas de negocio, los flujos y la API. El orden de implementación recomendado es:

1. Núcleo: autenticación, RBAC, catálogos, auditoría.
2. Expedientes georreferenciados + mapa.
3. Servicios, tarifas y facturación (lecturas → consumos → adeudos).
4. Cobranza (recibos, pagos, folios, corte de caja).
5. Egresos, donaciones, multas, inventario.
6. Órdenes de trabajo e infraestructura.
7. Reportes, mapas analíticos y configuraciones avanzadas.
