# BancoChat (Bansur)

Aplicación bancaria con un **asistente conversacional impulsado por IA** capaz de entender lo que el usuario necesita y generar, en tiempo real, la interfaz adecuada para resolverlo. Proyecto desarrollado para el reto **Banorte x Tec de Monterrey: "Interfaces que la IA construye en tiempo real"**.

Este repositorio agrupa los dos proyectos que componen la solución:

- **[`Back/`](Back)** — API REST (NestJS) con la lógica bancaria, autenticación y el agente de IA.
- **[`Front/`](Front)** — Aplicación web (Angular) con la banca en línea y el chat asistente.

---

## El problema

Las apps bancarias tradicionales obligan al usuario a navegar por múltiples pantallas fijas (cuentas, transferencias, metas de ahorro, presupuestos) aunque su necesidad real sea simple: *"quiero ahorrar $500 al mes para un viaje"* o *"¿cuánto gasté en comida este mes?"*. La interfaz es siempre la misma sin importar la intención, lo que genera fricción y obliga al usuario a aprender a usar la app en lugar de simplemente pedir lo que necesita.

## La solución

BancoChat resuelve esto con un **agente de IA que interpreta el lenguaje natural del usuario, ejecuta acciones bancarias reales y decide qué componente de interfaz mostrar según la intención detectada** — en vez de una UI fija, el usuario conversa y la aplicación construye la pantalla necesaria al vuelo (patrón **A2UI — Agent-to-UI**).

Flujo de referencia (meta de ahorro):

1. El usuario le dice al chat que quiere ahorrar para algo.
2. El agente simula el plan (`simulate_savings_plan`) y responde con texto + un componente de UI generado dinámicamente.
3. El usuario confirma dentro del propio chat.
4. El agente crea la meta en la base de datos real (`create_savings_goal`) y permite aportarle (`contribute_savings_goal`).

Toda acción sobre datos reales pasa siempre por el LLM (nunca se salta al backend directamente desde un click de la UI), cerrando el ciclo agente → herramienta → base de datos → UI.

### Arquitectura: LLM + MCP + A2UI

```
Usuario → POST /chat/sessions/:id/messages
              ↓
        ChatService — orquesta el loop con el LLM
              ↓
        LLM (Claude / Gemini, configurable) — decide qué tool llamar y cuándo terminar
              ↓
        Servidor MCP (Model Context Protocol) → Tools bancarias
              ↓
        Accounts / Transfers / Savings Goals / Budgets (capa real, Prisma + MySQL/MariaDB)
              ↓
        Respuesta = texto + uiSchema (A2UI) que el frontend renderiza dinámicamente
```

- **MCP real**: las herramientas que el agente puede usar (consultar cuentas, transferir, crear metas de ahorro, etc.) se exponen mediante el **SDK oficial de Model Context Protocol** (`@modelcontextprotocol/sdk`), no como simples funciones internas.
- **A2UI (Agent-to-UI)**: el agente no solo responde texto, también puede pedir que se renderice un componente concreto (tarjeta de meta de ahorro, lista de transacciones, estado de presupuesto, etc.) junto con sus datos. El frontend mantiene un *registro* de componentes Angular que sabe interpretar ese esquema y montarlo dinámicamente, sin necesidad de tocar el componente del chat cada vez que se agrega un bloque nuevo.
- **LLM intercambiable**: el proveedor de modelo (Anthropic Claude o Google Gemini) es configurable por variable de entorno, sin acoplar la lógica de negocio a un proveedor específico.

---

## Tecnologías utilizadas

### Backend (`Back/`)
| Categoría | Tecnología |
|---|---|
| Framework | [NestJS](https://nestjs.com/) (Node.js, TypeScript) |
| Base de datos | MySQL / MariaDB vía [Prisma ORM](https://www.prisma.io/) (`@prisma/adapter-mariadb`) |
| IA / LLM | [Anthropic Claude SDK](https://github.com/anthropics/anthropic-sdk-typescript) y [Google Gemini SDK](https://ai.google.dev/) (configurable) |
| Protocolo de herramientas del agente | [Model Context Protocol SDK](https://modelcontextprotocol.io/) |
| Autenticación | JWT (`@nestjs/jwt`, `passport-jwt`), contraseñas con `bcryptjs` |
| Validación | `class-validator`, `class-transformer`, `zod`, `joi` |
| Testing | [Vitest](https://vitest.dev/) (unitario y e2e) |
| Lint / formato | `oxlint`, `prettier` |
| Despliegue | [Railway](https://railway.app/) (`railway.json`) |

### Frontend (`Front/`)
| Categoría | Tecnología |
|---|---|
| Framework | [Angular 22](https://angular.dev/) — componentes standalone, signals, *zoneless* (sin `zone.js`) |
| Estilos | [Tailwind CSS v4](https://tailwindcss.com/) + [daisyUI](https://daisyui.com/) |
| Reactividad / HTTP | RxJS |
| Autenticación | `@auth0/angular-jwt` (JWT en `localStorage` + interceptor) |
| UI/UX | SweetAlert2 (alertas y confirmaciones) |
| Testing | Vitest (`@angular/build:unit-test`) |
| Contenedor | Docker (`Dockerfile` + `nginx.conf`) |

---

## Puesta en marcha

Se necesitan dos procesos corriendo en paralelo: el backend (API + agente) y el frontend (Angular).

### Requisitos previos

- Node.js `>=22.12.0`
- Una base de datos MySQL/MariaDB accesible (local o remota)
- Una API key de Anthropic (Claude) y/o de Google (Gemini)

### 1. Backend — `Back/`

```bash
cd Back
npm install
```

Copia `.env.example` a `.env` y completa los valores:

```bash
DATABASE_URL=""          # cadena de conexión a MySQL/MariaDB
JWT_SECRET=""            # secreto para firmar los tokens
JWT_EXPIRES_IN=28800
PORT=3000

LLM_PROVIDER="claude"    # "claude" o "gemini"

ANTHROPIC_API_KEY=""
ANTHROPIC_MODEL="claude-opus-5"

GEMINI_API_KEY=""
GEMINI_MODEL="gemini-2.5-flash"
```

Aplica las migraciones de Prisma y (opcionalmente) siembra datos de ejemplo:

```bash
npx prisma migrate deploy
npx prisma db seed
```

Levanta el servidor:

```bash
npm run start:dev     # modo watch, recomendado en desarrollo
# o
npm run start         # modo normal
# o
npm run start:prod    # aplica migraciones + corre el build de producción
```

La API queda disponible en `http://localhost:3000` (o el `PORT` configurado).

Comandos útiles:

```bash
npm run lint      # oxlint
npm run test      # pruebas unitarias (Vitest)
npm run test:e2e  # pruebas end-to-end
npm run test:cov  # cobertura
```

### 2. Frontend — `Front/`

```bash
cd Front
npm install
```

Configura la URL del backend en `src/environments/environment.development.ts` (desarrollo) y `environment.ts` (producción) mediante la propiedad `apiUrlBase`:

```ts
export const environment = {
  production: false,
  apiUrlBase: 'http://localhost:3000/', // debe apuntar al backend
};
```

Levanta la aplicación:

```bash
npm start   # ng serve → http://localhost:4200
```

Comandos útiles:

```bash
npm run build   # build de producción -> dist/
npm run watch   # build de desarrollo, se reconstruye con cada cambio
npm test        # pruebas unitarias (Vitest)
```

### 3. Usar la aplicación

1. Abre `http://localhost:4200` con el frontend y el backend corriendo.
2. Inicia sesión (usuario/contraseña contra `POST {apiUrlBase}auth/login`).
3. Explora las pantallas de cuentas, transacciones, transferencias, presupuestos y metas de ahorro.
4. Entra a **Asistente** y pídele en lenguaje natural algo como *"quiero ahorrar $3000 en 6 meses para un viaje"* o *"¿en qué gasté más este mes?"* — el agente responderá con texto y, cuando aplique, un componente interactivo generado dinámicamente.

---

## Estructura del repositorio

```
BancoChat/
├── Back/     # API NestJS + agente de IA (Claude/Gemini) + servidor MCP + Prisma
│   ├── docs/                  # contrato A2UI y referencia de la API
│   └── src/modules/           # auth, accounts, transfers, budgets, savings-goals, chat, categories...
└── Front/    # Aplicación Angular (banca en línea + chat asistente con A2UI)
    └── src/app/
        ├── core/               # guards, interceptors, servicios HTTP, stores (signals)
        ├── features/           # home, assistant (chat + A2UI), transfers, budgets, savings-goals...
        ├── layouts/            # auth-layout / main-layout
        └── shared/components/  # componentes genéricos reutilizables
```

Para más detalle técnico de cada proyecto, consulta el README específico de [`Back/`](Back/README.md) y de [`Front/`](Front/README.md), así como [`Back/docs/a2ui-contract.md`](Back/docs/a2ui-contract.md) (contrato entre agente y frontend) y [`Back/docs/api-reference.md`](Back/docs/api-reference.md) (referencia de endpoints).
