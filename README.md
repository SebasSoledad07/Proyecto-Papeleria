# Sistema de Inventario y Ventas para Papelería

Aplicación web para una papelería en Colombia: productos con código de barras (lector USB o cámara del celular), inventario con kárdex, caja con tirilla, y reportes de stock bajo y productos por agotarse. Todo en pesos colombianos.

## Estructura

```
papeleria/
├── AGENTS.md          reglas compartidas para OpenCode
├── docs/              negocio, reglas, contrato de la API, despliegue, plan
├── backend/           API REST: Java 21 · Spring Boot 4 · PostgreSQL
│   ├── AGENTS.md
│   └── docs/          arquitectura, base de datos, dependencias, consultas SQL
└── frontend/          SPA: React · TypeScript · Vite
    ├── AGENTS.md
    └── docs/          arquitectura, pantallas, integración con la API, escáner e impresión
```

## Cómo empezar (desarrollo)

```bash
# 1. Base de datos y API
cd backend
cp .env.example .env              # y cambiar las claves
docker compose up -d
./mvnw spring-boot:run -Dspring-boot.run.profiles=dev

# 2. Interfaz (en otra terminal)
cd frontend
npm install
npm run dev                       # http://localhost:5173
```

`./mvnw` requiere generar el wrapper de Maven una vez (`mvn wrapper:wrapper`). El frontend se crea con `npm create vite@latest` (ver `frontend/docs/dependencias-y-entorno.md`).

## Para leer primero

1. `docs/01-negocio-y-alcance.md` y `docs/02-reglas-de-negocio.md`
2. `docs/07-decisiones-pendientes.md` (hay decisiones que tomar antes de la fase 4)
3. `docs/04-contrato-api.md`
4. `docs/06-plan-y-aceptacion.md`

## Producción

Backend y PostgreSQL en un VPS con Docker Compose y Caddy; frontend en un hosting estático; dominio propio con `app.` y `api.`. Ver `docs/05-despliegue.md`.
