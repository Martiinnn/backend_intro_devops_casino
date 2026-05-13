# casino-backend

Backend del **Casino Online** — Experiencia 2 de la asignatura
**Introducción a Herramientas DevOps (ISY1101)**.

API REST en Node.js + Express con PostgreSQL como base de datos.

> ⚠️ **Este repositorio NO incluye `Dockerfile`, `docker-compose.yml`
> ni workflows de GitHub Actions.** Esos artefactos forman parte del
> entregable de la **Evaluación Parcial 2** y deben construirlos los
> estudiantes (frontend + backend + base de datos contenerizados,
> publicados en un registry y desplegados en EC2).

---

## Stack

- Node.js 20 (recomendado correr sobre `node:20-alpine`)
- Express 4
- PostgreSQL 16 (recomendado `postgres:16-alpine` con volumen nombrado)
- JWT para autenticación, bcryptjs para hashes
- `pg` como cliente de Postgres

---

## Estructura

```
casino-backend/
├── src/
│   ├── server.js                ← bootstrap Express + rutas
│   ├── db/
│   │   ├── pool.js              ← Pool de pg + esperarBD()
│   │   └── seed.js              ← usuarios demo (idempotente)
│   ├── middleware/
│   │   └── auth.js              ← JWT firmar / requiereAuth
│   ├── routes/
│   │   ├── auth.js              ← /api/auth/login | register
│   │   ├── users.js             ← /api/usuarios/me, depositar
│   │   ├── games.js             ← /api/juegos/{slots,roulette,blackjack}
│   │   └── transactions.js      ← /api/transacciones (historial)
│   └── games/
│       ├── slots.js
│       ├── roulette.js
│       └── blackjack.js
├── db/
│   └── init.sql                 ← esquema (lo monta Postgres en /docker-entrypoint-initdb.d)
├── package.json
├── .dockerignore
├── .gitignore
└── .env.example
```

---

## Variables de entorno

| Variable        | Default       | Descripción                                   |
|-----------------|---------------|-----------------------------------------------|
| `PORT`          | `3000`        | Puerto HTTP del servidor                      |
| `JWT_SECRET`    | `cambiame`    | Secreto de firma JWT (cambiar en producción)  |
| `JWT_EXPIRES_IN`| `8h`          | Vigencia del token                            |
| `DB_HOST`       | `localhost`   | Host de Postgres (`db` en docker-compose)     |
| `DB_PORT`       | `5432`        | Puerto Postgres                               |
| `DB_USER`       | `casino`      | Usuario Postgres                              |
| `DB_PASSWORD`   | `casino`      | Password Postgres                             |
| `DB_NAME`       | `casino_db`   | Base de datos                                 |
| `CORS_ORIGIN`   | `*`           | Lista CSV de orígenes permitidos              |

---

## Endpoints

### Autenticación

| Método | Ruta                  | Descripción                              |
|--------|-----------------------|------------------------------------------|
| POST   | `/api/auth/register`  | Registro `{ username, email, password }` |
| POST   | `/api/auth/login`     | Login `{ username, password }`           |

### Usuario autenticado (header `Authorization: Bearer <token>`)

| Método | Ruta                                  | Descripción                       |
|--------|---------------------------------------|-----------------------------------|
| GET    | `/api/usuarios/me`                    | Datos del usuario y saldo         |
| POST   | `/api/usuarios/me/depositar`          | `{ monto }` — recarga saldo demo  |
| GET    | `/api/transacciones?limit=50`         | Historial del usuario             |

### Juegos

| Método | Ruta                              | Descripción                                                    |
|--------|-----------------------------------|----------------------------------------------------------------|
| GET    | `/api/juegos`                     | Catálogo (slots, roulette, blackjack)                          |
| POST   | `/api/juegos/slots/jugar`         | `{ apuesta }` → `{ resultado, saldo }`                         |
| POST   | `/api/juegos/roulette/jugar`      | `{ apuestas:[{tipo,valor,monto}] }` → `{ resultado, saldo }`  |
| POST   | `/api/juegos/blackjack/iniciar`   | `{ apuesta }` → `{ sesionId, jugador, banca, ... }`            |
| POST   | `/api/juegos/blackjack/accion`    | `{ sesionId, accion: pedir/plantarse/doblar }`                 |

### Salud

| Método | Ruta       | Descripción                  |
|--------|------------|------------------------------|
| GET    | `/health`  | Estado del servidor + BD     |
| GET    | `/`        | Mensaje de bienvenida        |

---

## Usuarios demo (sembrados al arrancar)

| username   | password    | rol      | saldo inicial |
|------------|-------------|----------|---------------|
| `demo`     | `demo1234`  | jugador  | $5.000        |
| `jugador1` | `demo1234`  | jugador  | $1.000        |
| `admin`    | `admin1234` | admin    | $99.999       |

---

## Cómo correr en local (sin Docker)

Requisitos: Node 20 y un Postgres accesible.

```bash
cp .env.example .env          # ajustar credenciales
npm install
npm start
# API disponible en http://localhost:3000
```

---

## Cómo lo van a contenerizar (EP2)

El docente espera que ustedes:

1. Construyan un **Dockerfile multi-stage** (`builder` con `npm ci`,
   `runtime` `node:20-alpine` con usuario no root).
2. Definan en el `docker-compose.yml` los servicios `db`, `backend`
   (y agreguen el `frontend`) con:
   - `pg_data` como **named volume** para `/var/lib/postgresql/data`.
   - `./casino-backend/db/init.sql` montado en `/docker-entrypoint-initdb.d/`
     (recuerden: solo se ejecuta si el volumen está vacío).
   - `depends_on` con `condition: service_healthy` y un `healthcheck`
     en `db` (`pg_isready`).
   - Variables de entorno **inyectadas por compose**, sin hard-codear.
3. Configuren workflows en `.github/workflows/` que hagan
   `build → push (ECR) → deploy` en EC2 al hacer push a la rama
   correspondiente (en el **Ejercicio 2.5** se usa `main`; en la
   **EP2** la pauta oficial pide la rama `deploy`).

Lean la pauta oficial (`EP2_Instrucciones y Pauta_Encargo_Estudiante.pdf`)
para los criterios completos.

---

## Repositorio del frontend

[`casino-frontend`](../casino-frontend)

---

## CI/CD (GitHub Actions) - Rama `deploy`

Este repositorio incluye el workflow:

- `.github/workflows/deploy.yml`

Se ejecuta automaticamente con `push` a la rama `deploy` y sigue 3 etapas:

1. Build de imagen Docker.
2. Push al registry (Docker Hub).
3. Deploy automatico en EC2 por SSH.

### Tags de version en cada build

Cada ejecucion publica simultaneamente:

- `vX.Y.Z` (version tomada desde `package.json`)
- `latest`
- `${{ github.sha }}`

### Secrets requeridos

Configurar en `Settings > Secrets and variables > Actions`:

- `DOCKERHUB_USERNAME`
- `DOCKERHUB_TOKEN`
- `EC2_BACKEND_HOST`
- `EC2_BACKEND_USER`
- `EC2_BACKEND_SSH_KEY`
- `DB_HOST`
- `DB_PORT`
- `DB_USER`
- `DB_PASSWORD`
- `DB_NAME`
- `JWT_SECRET`
- `JWT_EXPIRES_IN`
- `CORS_ORIGIN`

No se almacenan credenciales en texto plano en el repositorio.
