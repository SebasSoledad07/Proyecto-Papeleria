# Sistema de Inventario y Ventas para Papelería

Aplicación web para una papelería en Colombia. Cubre el ciclo completo del negocio: catálogo de productos con código de barras (lector USB o cámara del celular), inventario con kárdex, compras, conteo físico, caja con tirilla, ventas con pagos mixtos y reportes con exportación a Excel/PDF. Todos los valores monetarios están en **pesos colombianos (COP) enteros** y las fechas se manejan en **America/Bogota**.

El repositorio principal agrupa dos proyectos independientes como **submódulos de Git**:

| Proyecto | Ruta | Repositorio |
|---|---|---|
| Backend — API REST | `backend/Inventario-Papeleria` | `SebasSoledad07/Inventario-Papeleria` |
| Frontend — SPA | `frontend/papeleria-frontend` | `SebasSoledad07/papeleria-frontend` |

> Para clonar todo el sistema hay que traer los submódulos:
>
> ```bash
> git clone --recurse-submodules <url-del-repositorio>
> # o, si ya está clonado:
> git submodule update --init --recursive
> ```

## Estado del proyecto

| Parte | Estado |
|---|---|
| **Backend** | Fases 1 a 5 completas: base/seguridad, catálogo, inventario, caja/ventas, clientes, reportes y panel. |
| **Frontend** | Fases 0 a 5 completas: autenticación, administración, catálogo, escáner, inventario, caja/ventas, reportes y preferencias. |
| **Despliegue (fase 6)** | Pendiente. El frontend ya trae artefactos (`deploy/Dockerfile` y `deploy/nginx.conf`); el backend todavía no incluye Dockerfile. |
| **Facturación electrónica DIAN** | Fuera del alcance de la versión 1. |

El estado detallado y las reglas de trabajo están en los `AGENTS.md` de cada submódulo.

## Arquitectura general

```
Proyecto-Papeleria/
├── README.md                     ← este documento
├── .gitmodules
├── backend/
│   └── Inventario-Papeleria/     API REST (Spring Boot + PostgreSQL)
│       ├── AGENTS.md             reglas y estado del backend
│       ├── docs/                 negocio, reglas, arquitectura, BD, plan
│       ├── src/main/java/        código de la API
│       ├── src/main/resources/   configuración y migraciones Flyway
│       ├── docker-compose.yml    PostgreSQL 16 en :5435
│       └── pom.xml
└── frontend/
    └── papeleria-frontend/       SPA (React + TypeScript + Vite)
        ├── AGENTS.md             reglas y estado del frontend
        ├── docs/                 arquitectura, pantallas, integración, escáner
        ├── deploy/               Dockerfile + nginx.conf
        ├── public/               landing.html, iconos, robots.txt
        ├── src/                  código de la SPA
        └── package.json
```

**Separación de responsabilidades:** el backend es la única fuente de verdad del negocio (stock, IVA, descuentos, totales, consecutivos, permisos). El frontend solo presenta datos y captura la intención del usuario. En desarrollo, Vite proxya `/api` hacia `http://localhost:8080`; en el contenedor de nginx se hace el mismo proxy hacia el servicio `backend`.

## Requisitos

| Herramienta | Versión | Uso |
|---|---|---|
| JDK (Temurin recomendado) | **21** | Compilar y ejecutar el backend |
| Maven | 3.9+ o el wrapper incluido | Dependencias y build del backend |
| PostgreSQL | **16** | Base de datos (Docker o instalación directa) |
| Docker Desktop | Reciente | PostgreSQL local y pruebas de integración |
| Node.js | **≥ 22.12** (LTS) | Frontend (Vite 8) |
| npm | El incluido con Node | Dependencias del frontend |
| Git | Reciente | Control de versiones |

Hardware sugerido para la operación real: lector de código de barras USB/Bluetooth en modo HID con sufijo Enter, impresora térmica de tirillas de 58 u 80 mm y, opcionalmente, impresora de etiquetas y cajón monedero.

## Puesta en marcha (desarrollo)

### 1. Base de datos

```bash
cd backend/Inventario-Papeleria
docker compose up -d
```

El contenedor `papeleria-db` expone PostgreSQL en **`localhost:5435`** y crea la base `papeleria_db` con el usuario `app_papel`.

`docker-compose.yml` lee un archivo `.env` opcional. Las variables son:

```env
DB_NOMBRE=papeleria_db
DB_USUARIO=app_papel
DB_CLAVE=una-clave-local
```

> **Ojo:** los valores por defecto del `docker-compose.yml` y de `application.properties` no coinciden. Define `DB_CLAVE` de forma explícita y usa el mismo valor al arrancar la aplicación.

### 2. Backend

El backend lee la configuración de variables de entorno (Spring Boot **no** lee `.env` por sí solo). En PowerShell:

```powershell
cd backend/Inventario-Papeleria
$env:DB_URL = "jdbc:postgresql://localhost:5435/papeleria_db"
$env:DB_USUARIO = "app_papel"
$env:DB_CLAVE = "una-clave-local"
$env:ADMIN_INICIAL_USUARIO = "admin"
$env:ADMIN_INICIAL_CLAVE = "cambia-esta-clave"
.\mvnw.cmd spring-boot:run
```

En Bash/macOS/Linux:

```bash
cd backend/Inventario-Papeleria
export DB_URL="jdbc:postgresql://localhost:5435/papeleria_db"
export DB_USUARIO="app_papel"
export DB_CLAVE="una-clave-local"
export ADMIN_INICIAL_USUARIO="admin"
export ADMIN_INICIAL_CLAVE="cambia-esta-clave"
./mvnw spring-boot:run
```

La API queda en **`http://localhost:8080`**. Al arrancar, Flyway aplica las migraciones de `src/main/resources/db/migration`.

**Administrador inicial:** si la tabla de usuarios está vacía y existen `ADMIN_INICIAL_USUARIO` y `ADMIN_INICIAL_CLAVE`, se crea un usuario `ADMIN`. Si faltan, no se crea ninguno y se registra una advertencia. La clave debe cambiarse en el primer ingreso.

Documentación interactiva de la API (pública):

- Swagger UI: `http://localhost:8080/swagger-ui.html`
- OpenAPI JSON: `http://localhost:8080/v3/api-docs`

### 3. Frontend

En otra terminal:

```bash
cd frontend/papeleria-frontend
npm install
npm run dev
```

La SPA queda en **`http://localhost:5173`** y usa el proxy de Vite hacia `http://localhost:8080` para `/api`. El backend debe estar corriendo.

Para apuntar a otra URL de API, define `VITE_API_URL` antes de compilar o arrancar:

```env
# frontend/papeleria-frontend/.env.local
VITE_API_URL=http://localhost:8080
```

Si queda vacía, se usa el mismo origen (proxy de Vite o de nginx).

---

# Backend

## Stack

- **Java 21** + **Spring Boot 4.1.1**
- **PostgreSQL 16** + **Flyway** (migraciones versionadas)
- **Spring Data JPA/Hibernate** (`ddl-auto=validate`, `open-in-view=false`)
- **Spring JDBC (`JdbcClient`)** para los reportes (SQL directo, sin JPA)
- **Spring Security** con sesión HTTP (`JSESSIONID`), BCrypt y roles
- **Bean Validation** para los DTO de entrada
- **ZXing 3.5.4** para leer códigos desde foto y generar etiquetas
- **Apache POI 5.5.1** (Excel) y **OpenPDF 3.0.5** (PDF) para exportaciones
- **Springdoc OpenAPI 3.1.1** (Swagger UI)
- **Lombok**, **ArchUnit 1.5.0** y **Testcontainers 2.x** para pruebas
- Maven Wrapper incluido (`mvnw` / `mvnw.cmd`)

## Estructura del código

Paquete base: `com.soledad.inventariopapeleria`. Organización **por funcionalidad**, con capas `api (Controller + DTO) → servicio (@Transactional) → dominio → repositorio`.

```
src/main/java/com/soledad/inventariopapeleria/
├── InventarioPapeleriaApplication.java   fija America/Bogota (RN-39)
├── compartido/                           Pesos (formato COP, RN-02/03)
├── error/                                excepciones + GlobalHandler
├── usuario/                              usuarios, roles, seguridad, admin inicial
├── auditoria/                            registro de auditoría
├── configuracion/                        parámetros del negocio
├── producto/                             productos, categorías, IVA, unidades, precios
├── proveedor/                            proveedores
├── codigo/                               normalización EAN, lector ZXing, etiquetas
├── catalogo/                             Open Products Facts (sugerencias, falla suave)
├── cliente/                              clientes
├── inventario/                           stock, kárdex, compras, conteo físico
├── venta/                                cajas, ventas, pagos, consecutivos, tirilla
└── reporte/                              reportes SQL + exportación Excel/PDF
```

Dependencias permitidas entre módulos: `venta → inventario → producto`. El módulo `producto` nunca llama a `inventario` ni a `venta`; `reporte` solo lee por SQL. **Solo `InventarioService` modifica `productos.stock`** (RN-20) y lo hace con `UPDATE ... WHERE stock >= :cantidad` (RN-19). Las reglas se verifican con ArchUnit en las pruebas.

## Funcionalidades

- Autenticación por sesión, roles `ADMIN`, `BODEGUERO` y `CAJERO`, y auditoría.
- Usuarios, configuración, categorías y proveedores.
- Productos: CRUD, activar/inactivar, búsqueda por nombre (`unaccent` + `pg_trgm`) y por código, historial de precios, generación de código interno EAN-13.
- Códigos: decodificación de fotos con ZXing y etiquetas PNG.
- Catálogo externo: sugerencia de nombre/marca/categoría por código (si falla, no bloquea el alta).
- Inventario: compras con costo promedio ponderado, ajustes positivos/negativos con motivo, kárdex y conteo físico con diferencias.
- Caja: apertura con base, cierre con cuadre y diferencia, una caja abierta por usuario.
- Ventas: confirmación atómica (consecutivo, detalle, pagos, IVA, descuentos y descuento de stock), consulta con filtros, anulación y tirilla HTML.
- Clientes: creación y consulta.
- Reportes: stock bajo, agotados, por agotarse, pedido por proveedor, ventas por periodo, productos más vendidos/sin movimiento, utilidad por periodo/categoría/producto, valor del inventario, cierres de caja y panel. Exportación a Excel/PDF.
- Zona horaria `America/Bogota` en JVM, Hibernate y Jackson.

## API REST

Casi todas las rutas están en **singular**. Roles: `A` = ADMIN, `B` = BODEGUERO, `C` = CAJERO, **Todos** = cualquier usuario autenticado.

### Autenticación, usuarios y configuración

| Método | Ruta | Descripción | Rol |
|---|---|---|---|
| POST | `/api/auth/login` | Iniciar sesión (cookie de sesión) | Público |
| POST | `/api/auth/logout` | Cerrar sesión | Público |
| GET | `/api/auth/sesion` | Usuario y rol de la sesión actual | Autenticado |
| GET | `/api/usuario` | Listar usuarios | A |
| POST | `/api/usuario` | Crear usuario | A |
| PUT | `/api/usuario/{id}` | Editar usuario | A |
| DELETE | `/api/usuario/{id}` | Inactivar/eliminar usuario | A |
| PUT | `/api/usuario/{id}/clave` | Cambiar clave | A |
| GET | `/api/configuracion` | Obtener configuración | A |
| PUT | `/api/configuracion` | Actualizar configuración | A |

### Catálogo

| Método | Ruta | Descripción | Rol |
|---|---|---|---|
| GET | `/api/categoria?soloActivas=true` | Listar categorías | Todos |
| POST | `/api/categoria` | Crear categoría | A |
| PUT | `/api/categoria/{id}` | Editar categoría | A |
| DELETE | `/api/categoria/{id}` | Inactivar categoría | A |
| GET | `/api/proveedor` | Listar proveedores | Todos |
| POST | `/api/proveedor` | Crear proveedor | A/B |
| PUT | `/api/proveedor/{id}` | Editar proveedor | A/B |
| DELETE | `/api/proveedor/{id}` | Inactivar proveedor | A/B |
| GET | `/api/producto` | Listar productos activos | Todos |
| GET | `/api/producto/buscar?q=` | Buscar por nombre | Todos |
| GET | `/api/producto/codigo/{codigo}` | Buscar por código de barras | Todos |
| POST | `/api/producto` | Crear producto | A/B |
| PUT | `/api/producto/{id}` | Editar producto | A/B |
| PUT | `/api/producto/{id}/precio` | Actualizar precio (historial) | A/B |
| PUT | `/api/producto/{id}/activar` | Activar producto | A/B |
| DELETE | `/api/producto/{id}` | Inactivar producto | A/B |
| GET | `/api/catalogo/codigo/{codigo}` | Sugerencia desde catálogo externo | Todos |
| POST | `/api/codigos/decodificar` | Decodificar código desde foto (`multipart`) | Todos |
| GET | `/api/codigos/{codigo}/etiqueta` | Generar etiqueta PNG | Todos |

### Inventario

| Método | Ruta | Descripción | Rol |
|---|---|---|---|
| POST | `/api/inventario/{id}/ajuste-positivo` | Ajuste de entrada con motivo | A/B |
| POST | `/api/inventario/{id}/ajuste-negativo` | Ajuste de salida con motivo | A/B |
| GET | `/api/inventario/productos/{id}/kardex` | Movimientos de un producto | A/B |
| POST | `/api/inventario/conteo-fisico` | Aplicar conteo físico | A/B |
| POST | `/api/inventario/compras` | Registrar compra | A/B |
| GET | `/api/inventario/compras/{id}` | Consultar compra | A/B |

### Caja, ventas y clientes

| Método | Ruta | Descripción | Rol |
|---|---|---|---|
| POST | `/api/caja/abrir` | Abrir caja con base en efectivo | A/C |
| GET | `/api/caja/abierta` | Caja abierta del usuario | A/C |
| GET | `/api/caja/{id}` | Consultar caja | A/C |
| POST | `/api/caja/{id}/cerrar` | Cerrar caja y calcular diferencia | A/C |
| POST | `/api/venta` | Confirmar venta y devolver totales oficiales | A/C |
| GET | `/api/venta` | Buscar ventas con filtros | A/C |
| GET | `/api/venta/{id}` | Detalle de venta | A/C |
| POST | `/api/venta/{id}/anular` | Anular venta con motivo | A/C |
| GET | `/api/venta/{id}/tirilla` | Tirilla HTML (58/80 mm) | A/C |
| POST | `/api/cliente` | Crear cliente | A/C |
| GET | `/api/cliente/{id}` | Consultar cliente | A/C |

### Reportes

| Método | Ruta | Descripción | Rol |
|---|---|---|---|
| GET | `/api/reporte/stock-bajo` | Productos bajo el mínimo | Todos |
| GET | `/api/reporte/agotados` | Productos sin stock | Todos |
| GET | `/api/reporte/por-agotarse` | Proyección de agotamiento | Todos |
| GET | `/api/reporte/pedido-proveedor` | Pedido sugerido por proveedor | Todos |
| GET | `/api/reporte/ventas-periodo?desde=&hasta=&agrupacion=` | Ventas agrupadas | Todos |
| GET | `/api/reporte/productos-mas-vendidos?desde=&hasta=&limite=` | Ranking de ventas | Todos |
| GET | `/api/reporte/productos-sin-movimiento?dias=` | Productos sin rotación | Todos |
| GET | `/api/reporte/utilidad-periodo?desde=&hasta=` | Utilidad por periodo | A |
| GET | `/api/reporte/utilidad-categoria?desde=&hasta=` | Utilidad por categoría | A |
| GET | `/api/reporte/utilidad-producto?desde=&hasta=` | Utilidad por producto | A |
| GET | `/api/reporte/valor-inventario` | Valor del inventario | A |
| GET | `/api/reporte/cierres-caja?desde=&hasta=` | Cierres de caja | Todos |
| GET | `/api/reporte/panel` | Indicadores del día y alertas | Todos |
| GET | `/api/reporte/exportar?reporte=&formato=&desde=&hasta=` | Exportar a Excel (`xlsx`) o PDF (`pdf`) | Todos |

> Los reportes devuelven `List<Map<String,Object>>` / `Map<String,Object>` con nombres de columna en `snake_case`; no hay DTO tipado para ellos.

## Seguridad

- Autenticación por **sesión HTTP** con cookie `JSESSIONID`; **no se usa JWT**.
- **CSRF deshabilitado** en `SecurityConfig`.
- Claves cifradas con **BCrypt** (RN-43); nunca se guardan en claro.
- Errores de autenticación en JSON: `401` sin sesión, `403` sin permiso.
- Matriz de permisos: `ADMIN` todo; `BODEGUERO` escribe productos/proveedores y todo `/api/inventario/**`; `CAJERO` opera caja, ventas y clientes; categorías, usuarios y configuración son solo de `ADMIN`.
- Las lecturas GET de producto, proveedor y categoría están disponibles para cualquier autenticado.
- La landing y los estáticos del backend no se sirven desde este proyecto; el frontend es independiente.

## Base de datos

- **16 tablas** + la vista `v_estado_stock`, creadas por `V1__esquema_inicial.sql` y `V2__vista_estado_stock.sql`.
- Tablas principales: `usuarios`, `configuracion`, `auditoria`, `categorias`, `proveedores`, `productos`, `historial_precios`, `movimientos_inventario`, `compras`, `detalle_compra`, `clientes`, `cajas`, `ventas`, `detalle_venta`, `pagos_venta`, `consecutivos`.
- Restricciones e índices garantizan stock no negativo, códigos únicos, búsqueda por trigramas y consistencia de estados.
- El esquema **no** se genera desde JPA: Flyway es la única vía de cambio.

## Pruebas del backend

```bash
cd backend/Inventario-Papeleria
./mvnw test
```

- **12 archivos de prueba** (38 `@Test` + 1 `@ParameterizedTest`).
- Paquetes: `unitarias` (calculadoras y generadores puros), `arquitectura` (ArchUnit) y el test de contexto de Spring.
- El test de contexto requiere PostgreSQL real en `localhost:5435`; no se usa H2.
- No hay linter ni checkstyle: la verificación es compilación + ArchUnit + pruebas.

## Documentación del backend

| Tema | Archivo |
|---|---|
| Reglas de negocio (RN-01 a RN-46) | [`docs/02-reglas-de-negocio.md`](backend/Inventario-Papeleria/docs/02-reglas-de-negocio.md) |
| Requerimientos funcionales y no funcionales | [`docs/03-requerimientos.md`](backend/Inventario-Papeleria/docs/03-requerimientos.md) |
| Arquitectura y dependencias entre módulos | [`docs/04-arquitectura.md`](backend/Inventario-Papeleria/docs/04-arquitectura.md) |
| Modelo de datos | [`docs/05-base-de-datos.md`](backend/Inventario-Papeleria/docs/05-base-de-datos.md) |
| Dependencias y entorno | [`docs/06-dependencias-y-entorno.md`](backend/Inventario-Papeleria/docs/06-dependencias-y-entorno.md) |
| Plan y criterios de aceptación | [`docs/07-plan-y-aceptacion.md`](backend/Inventario-Papeleria/docs/07-plan-y-aceptacion.md) |
| Decisiones pendientes | [`docs/08-decisiones-pendientes.md`](backend/Inventario-Papeleria/docs/08-decisiones-pendientes.md) |
| Consultas SQL de reportes | [`docs/consultas-sql.md`](backend/Inventario-Papeleria/docs/consultas-sql.md) |
| Reglas de trabajo para agentes | [`AGENTS.md`](backend/Inventario-Papeleria/AGENTS.md) |

---

# Frontend

## Stack

- **React 19** + **TypeScript ~6** + **Vite 8**
- **Tailwind CSS v4** + **shadcn/ui sobre Base UI** (no Radix)
- **React Router DOM 7** (SPA con `createBrowserRouter`)
- **TanStack Query 5** para estado del servidor; `useState` para estado de interfaz
- **React Hook Form 7 + Zod 4** para formularios
- **barcode-detector 3** para lectura con cámara
- **Recharts 3** para gráficas
- **Vitest 5 + Testing Library + jsdom + MSW 2** para pruebas
- Tipografías locales con `@fontsource-variable`: **Fraunces** (títulos), **Geist** (interfaz) y **Geist Mono** (códigos)

## Estructura del código

Organización **por característica**, no por tipo de archivo:

```
src/
├── api/                  cliente.ts, errores.ts, tipos.ts (manual)
├── lib/                  formato.ts (único lugar de formato COP), utils.ts
├── escaner/              CampoCodigo, CamaraEscaner, FotoEscaner, bip
├── rutas/                router, navegación por rol, RutaProtegida
├── componentes/
│   ├── ui/               componentes generados por shadcn (editables)
│   └── comunes/          avisos, campos, dinero, carga, sin conexión
├── caracteristicas/
│   ├── autenticacion/    login, sesión, logout
│   ├── productos/        listado, alta, edición, códigos
│   ├── inventario/       compras, ajustes, conteo, kárdex
│   ├── caja/             apertura, cobro, cierre, ventas
│   ├── reportes/         lista y tabla genérica de reportes
│   ├── usuarios/         CRUD y cambio de clave
│   ├── categorias/       CRUD
│   ├── proveedores/      CRUD
│   ├── configuracion/    parámetros del negocio
│   ├── panel/            indicadores del día
│   ├── preferencias/     ajustes por equipo (localStorage)
│   └── errores/          404 y sin permiso
└── pruebas/              setup, servidor MSW y handlers
```

Convenciones: componentes y textos en **español**; `PascalCase` para componentes y `camelCase` para hooks/funciones; un componente por archivo; alias `@/*` → `./src`.

## Rutas y pantallas

| Ruta | Pantalla | Roles |
|---|---|---|
| `/login` | Inicio de sesión | Público |
| `/` | Panel con indicadores y alertas de stock | A/B/C |
| `/productos` | Listado y búsqueda de productos | A/B/C |
| `/productos/nuevo` | Alta de producto con escaneo y sugerencia | A/B |
| `/productos/:id/kardex` | Kárdex del producto | A/B |
| `/inventario/compras` | Registro de compras | A/B |
| `/inventario/ajustes` | Ajustes de inventario con motivo | A/B |
| `/inventario/conteo` | Conteo físico y diferencias | A/B |
| `/caja` | Caja: carrito, descuentos, pagos mixtos y tirilla | A/C |
| `/caja/cierre` | Cierre y cuadre de caja | A/C |
| `/ventas` | Consulta de ventas | A/C |
| `/ventas/:id` | Detalle, reimpresión y anulación | A/C |
| `/reportes` | Índice de reportes por rol | A/B |
| `/reportes/:nombre` | Tabla de un reporte con filtros y exportación | A/B |
| `/usuarios` | Gestión de usuarios | A |
| `/categorias` | Gestión de categorías | A |
| `/proveedores` | Gestión de proveedores | A |
| `/configuracion` | Parámetros del negocio | A |
| `/preferencias` | Preferencias del equipo (localStorage) | A/B/C |
| `/404` | Página no encontrada | Público |

El menú lateral se arma desde `src/rutas/navegacion.ts` y oculta las opciones que el rol no puede usar. Es comodidad de interfaz: **el backend vuelve a validar cada petición**.

## Funcionalidades

- Login/logout con cookie de sesión y redirección por rol.
- CRUD de usuarios, categorías y proveedores; configuración del negocio.
- Productos: búsqueda, alta con escaneo, sugerencia desde catálogo externo, edición, cambio de precio, activación/inactivación, kárdex y etiquetas.
- Escáner: lector USB/Bluetooth (modo teclado), cámara y decodificación de fotos; bip configurable.
- Inventario: compras, ajustes con motivo, conteo físico, kárdex y visibilidad de costos según rol.
- Caja: apertura, carrito por escaneo, descuentos, pagos mixtos, cambio, tirilla de 58/80 mm, cierre con cuadre.
- Ventas: consulta con filtros, detalle, reimpresión de tirilla y anulación.
- Reportes: diez reportes con filtros de fechas, tabla genérica y exportación Excel/PDF.
- Panel con indicadores del día, gráficas y alertas de stock.
- Preferencias por equipo: ancho de tirilla, impresión automática, sonido y cámara (guardadas en `localStorage`).
- Landing comercial independiente en `public/landing.html` (`/landing.html`), fuera del router de React.

## Scripts

```bash
npm install        # dependencias
npm run dev        # servidor de desarrollo en http://localhost:5173
npm run build      # typecheck (tsc -b) + build de producción → dist/
npm run lint       # ESLint
npm test           # Vitest en modo run (jsdom + MSW)
npm run preview    # servir el build localmente
```

## Pruebas del frontend

- **9 archivos de prueba / 46 pruebas** con Vitest, Testing Library, jsdom y MSW.
- MSW usa `onUnhandledRequest: "error"`: si se agrega una llamada a un endpoint nuevo, hay que registrar su handler en `src/pruebas/mocks/handlers.ts`.
- El catálogo mock es mutable y se reinicia con `reiniciarCatalogo()`.

## Diseño

- Tema **«Papel y tinta»**: fondo cálido y acento terracota; títulos en Fraunces, interfaz en Geist y códigos en Geist Mono.
- Tailwind v4 se configura en CSS (`@theme`), sin `tailwind.config.js`. El tema vive en `src/App.css`.
- Sin fuentes ni recursos remotos: todo se empaqueta localmente.

## Despliegue del frontend

Los artefactos están en `deploy/`:

- `deploy/Dockerfile`: build multi-etapa con Node 22 y servicio con nginx.
- `deploy/nginx.conf`: sirve la SPA con fallback a `index.html` y proxya `/api/` hacia `http://backend:8080/api/`.

```bash
cd frontend/papeleria-frontend
npm run build                       # genera dist/
docker build -f deploy/Dockerfile -t papeleria-frontend .
```

`VITE_API_URL` se resuelve en **tiempo de build**. Si se deja vacía y se usa nginx con el proxy, la SPA consume la API en el mismo origen.

## Documentación del frontend

| Tema | Archivo |
|---|---|
| Arquitectura del frontend | [`docs/arquitectura.md`](frontend/papeleria-frontend/docs/arquitectura.md) |
| Pantallas y flujos | [`docs/pantallas-y-flujos.md`](frontend/papeleria-frontend/docs/pantallas-y-flujos.md) |
| Integración con la API | [`docs/integracion-api.md`](frontend/papeleria-frontend/docs/integracion-api.md) |
| Escáner, cámara, tirilla y etiquetas | [`docs/escaner-e-impresion.md`](frontend/papeleria-frontend/docs/escaner-e-impresion.md) |
| Dependencias y entorno | [`docs/dependencias-y-entorno.md`](frontend/papeleria-frontend/docs/dependencias-y-entorno.md) |
| Plan del frontend | [`docs/plan-frontend.md`](frontend/papeleria-frontend/docs/plan-frontend.md) |
| Contrato de API (borrador) | [`docs/04-contrato-api.md`](frontend/papeleria-frontend/docs/04-contrato-api.md) |
| Reglas de trabajo para agentes | [`AGENTS.md`](frontend/papeleria-frontend/AGENTS.md) |

> Algunos documentos del frontend están desactualizados respecto al código (por ejemplo, mencionan plurales, CSRF, `openapi-fetch` o Radix). **La fuente de verdad son los controllers del backend y el código actual**; los `AGENTS.md` recogen las diferencias conocidas.

---

# Integración frontend ↔ backend

- **Base URL:** en desarrollo `/api` vía proxy de Vite; en producción `/api` vía nginx o `VITE_API_URL` con la URL absoluta.
- **Sesión:** todas las peticiones usan `credentials: "include"` (ya configurado en `src/api/cliente.ts`). No se guardan tokens ni claves en `localStorage`.
- **CSRF:** deshabilitado en el backend; el frontend no envía token CSRF.
- **Errores:** el backend devuelve JSON con `mensaje`; el frontend lo traduce con `mensajeDeError`. Un `403` se muestra como «No tienes permiso».
- **Dinero:** enteros en COP. El único formateador es `src/lib/formato.ts` → `$ 12.500`. No se usa `parseFloat` sobre separadores de miles.
- **Servidor como verdad:** no existe un endpoint de cálculo previo. La venta se calcula al confirmar con `POST /api/venta` y los totales oficiales llegan en la respuesta. Localmente solo se muestra el subtotal `precioVenta × cantidad`.
- **Ventas:** no hay reintento automático ni soporte de `Idempotency-Key`; el botón de cobro se deshabilita mientras la petición está en curso.
- **Fechas:** `America/Bogota`, formato colombiano, en backend y frontend.
- **Tirilla:** «Comprobante de venta — no es factura electrónica».
- **CORS:** no hay configuración explícita. Se recomienda mantener frontend y API en el mismo origen mediante proxy (desarrollo y nginx) o agregar CORS en el backend si se publican en dominios distintos.

## Notas y limitaciones conocidas

- **Decisiones pendientes del backend:** D-1 (prorrateo de descuentos), D-2 (anulación con caja cerrada), D-4 (administrador inicial), D-5 (impresión de tirilla) y D-6 (autorización de descuentos). Están implementadas de forma provisional y marcadas con `PENDIENTE-DECISION-n`.
- **Decisiones del frontend no registradas en el backend:** D-7 (presentaciones por caja, omitida en v1), D-9 (ocultar costos por rol) y D-10.
- **Desviación conocida:** `ProductoService.crear` escribe `stockInicial` directamente en `producto` en lugar de pasar por `InventarioService`; hay que decidir si se corrige para respetar RN-20.
- **Pruebas de integración:** el paquete `integracion` con Testcontainers aún no existe, aunque las dependencias están en el `pom.xml`.
- **Contrato:** `docs/04-contrato-api.md` y `docs/integracion-api.md` del frontend son borradores incompletos; verificar siempre contra los `*Controller.java`.
- **Fuera de alcance v1:** venta sin conexión, facturación electrónica DIAN, tienda en línea, varias sedes, modo oscuro e i18n.
