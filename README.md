# BancoChat

Aplicacion bancaria con un **asistente conversacional impulsado por IA** capaz de entender lo que el usuario necesita y generar, en tiempo real, la interfaz adecuada para resolverlo. Proyecto desarrollado para el reto **Banorte x Tec de Monterrey: "Interfaces que la IA construye en tiempo real"**.

Este repositorio agrupa los dos proyectos que componen la solucion:

- **[Back/](Back)** - API REST (NestJS) con la logica bancaria, autenticacion y el agente de IA.
- **[Front/](Front)** - Aplicacion web (Angular) con la banca en linea y el chat asistente.

---

## El problema

Las apps bancarias tradicionales obligan al usuario a navegar por multiples pantallas fijas (cuentas, transferencias, metas de ahorro, presupuestos) aunque su necesidad real sea simple: "quiero ahorrar $500 al mes para un viaje" o "cuanto gaste en comida este mes?". La interfaz es siempre la misma sin importar la intencion, lo que genera friccion y obliga al usuario a aprender a usar la app en lugar de simplemente pedir lo que necesita.

## La solucion

BancoChat resuelve esto con un **agente de IA que interpreta el lenguaje natural del usuario, ejecuta acciones bancarias reales y decide que componente de interfaz mostrar segun la intencion detectada** - en vez de una UI fija, el usuario conversa y la aplicacion construye la pantalla necesaria al vuelo (patron **A2UI - Agent-to-UI**).

Flujo de referencia (meta de ahorro):

1. El usuario le dice al chat que quiere ahorrar para algo.
2. El agente simula el plan (simulate_savings_plan) y responde con texto + un componente de UI generado dinamicamente.
3. El usuario confirma dentro del propio chat.
4. El agente crea la meta en la base de datos real (create_savings_goal) y permite aportarle (contribute_savings_goal).

Toda accion sobre datos reales pasa siempre por el LLM (nunca se salta al backend directamente desde un click de la UI), cerrando el ciclo agente -> herramienta -> base de datos -> UI.

### Arquitectura: LLM + MCP + A2UI

```
Usuario -> POST /chat/sessions/:id/messages
              |
        ChatService - orquesta el loop con el LLM
              |
        LLM (Claude, Anthropic) - decide que tool llamar y cuando terminar
              |
        Servidor MCP (Model Context Protocol) -> Tools bancarias
              |
        Accounts / Transfers / Savings Goals / Budgets (capa real, Prisma + MySQL/MariaDB)
              |
        Respuesta = texto + uiSchema (A2UI) que el frontend renderiza dinamicamente
```

- **MCP real**: las herramientas que el agente puede usar (consultar cuentas, transferir, crear metas de ahorro, etc.) se exponen mediante el **SDK oficial de Model Context Protocol** (@modelcontextprotocol/sdk), no como simples funciones internas.
- **A2UI (Agent-to-UI)**: el agente no solo responde texto, tambien puede pedir que se renderice un componente concreto (tarjeta de meta de ahorro, lista de transacciones, estado de presupuesto, etc.) junto con sus datos. El frontend mantiene un registro de componentes Angular que sabe interpretar ese esquema y montarlo dinamicamente, sin necesidad de tocar el componente del chat cada vez que se agrega un bloque nuevo.
- **LLM**: el agente usa **Claude (Anthropic)** como modelo de lenguaje mediante su SDK oficial.

---

## Tecnologias utilizadas

### Backend (Back/)

| Categoria | Tecnologia |
|---|---|
| Framework | NestJS (Node.js, TypeScript) - https://nestjs.com/ |
| Base de datos | MySQL / MariaDB via Prisma ORM (@prisma/adapter-mariadb) - https://www.prisma.io/ |
| IA / LLM | Anthropic Claude SDK - https://github.com/anthropics/anthropic-sdk-typescript |
| Protocolo de herramientas del agente | Model Context Protocol SDK - https://modelcontextprotocol.io/ |
| Autenticacion | JWT (@nestjs/jwt, passport-jwt), contrasenas con bcryptjs |
| Validacion | class-validator, class-transformer, zod, joi |
| Testing | Vitest (unitario y e2e) - https://vitest.dev/ |
| Lint / formato | oxlint, prettier |
| Despliegue | Railway (railway.json) - https://railway.app/ |

### Frontend (Front/)

| Categoria | Tecnologia |
|---|---|
| Framework | Angular 22 (https://angular.dev/) - componentes standalone, signals, zoneless (sin zone.js) |
| Estilos | Tailwind CSS v4 (https://tailwindcss.com/) + daisyUI (https://daisyui.com/) |
| Reactividad / HTTP | RxJS |
| Autenticacion | @auth0/angular-jwt (JWT en localStorage + interceptor) |
| UI/UX | SweetAlert2 (alertas y confirmaciones) |
| Testing | Vitest (@angular/build:unit-test) |
| Contenedor | Docker (Dockerfile + nginx.conf) |

---

## Puesta en marcha

Se necesitan dos procesos corriendo en paralelo: el backend (API + agente) y el frontend (Angular).

El usuario y contraseña para el acceso a la página son: 
mlopez  - password123

### Requisitos previos

- Node.js >=22.12.0
- Una base de datos MySQL/MariaDB accesible (local o remota)
- Una API key de Anthropic (Claude)

### 1. Backend - Back/

```bash
cd Back
npm install
```

Copia .env.example a .env y completa los valores:

```bash
DATABASE_URL=""          # cadena de conexion a MySQL/MariaDB
JWT_SECRET=""            # secreto para firmar los tokens
JWT_EXPIRES_IN=28800
PORT=3000

LLM_PROVIDER="claude"

ANTHROPIC_API_KEY=""
ANTHROPIC_MODEL="claude-opus-5"
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
npm run start:prod    # aplica migraciones + corre el build de produccion
```

La API queda disponible en http://localhost:3000 (o el PORT configurado).

Comandos utiles:

```bash
npm run lint      # oxlint
npm run test      # pruebas unitarias (Vitest)
npm run test:e2e  # pruebas end-to-end
npm run test:cov  # cobertura
```

### 2. Frontend - Front/

```bash
cd Front
npm install
```

Configura la URL del backend en src/environments/environment.development.ts (desarrollo) y environment.ts (produccion) mediante la propiedad apiUrlBase:

```ts
export const environment = {
  production: false,
  apiUrlBase: 'http://localhost:3000/', // debe apuntar al backend
};
```

Levanta la aplicacion:

```bash
npm start   # ng serve -> http://localhost:4200
```

Comandos utiles:

```bash
npm run build   # build de produccion -> dist/
npm run watch   # build de desarrollo, se reconstruye con cada cambio
npm test        # pruebas unitarias (Vitest)
```

### 3. Usar la aplicacion

1. Abre http://localhost:4200 con el frontend y el backend corriendo.
2. Inicia sesion (usuario/contrasena contra POST {apiUrlBase}auth/login).
3. Explora las pantallas de cuentas, transacciones, transferencias, presupuestos y metas de ahorro.
4. Entra a **Asistente** y pidele en lenguaje natural algo como "quiero ahorrar $3000 en 6 meses para un viaje" o "en que gaste mas este mes?" - el agente respondera con texto y, cuando aplique, un componente interactivo generado dinamicamente.

---

## Estructura del repositorio

```
BancoChat/
|-- Back/     # API NestJS + agente de IA (Claude) + servidor MCP + Prisma
|   |-- docs/                  # contrato A2UI y referencia de la API
|   `-- src/modules/           # auth, accounts, transfers, budgets, savings-goals, chat, categories...
`-- Front/    # Aplicacion Angular (banca en linea + chat asistente con A2UI)
    `-- src/app/
        |-- core/               # guards, interceptors, servicios HTTP, stores (signals)
        |-- features/           # home, assistant (chat + A2UI), transfers, budgets, savings-goals...
        |-- layouts/            # auth-layout / main-layout
        `-- shared/components/  # componentes genericos reutilizables
```

Para mas detalle tecnico de cada proyecto, consulta el README especifico de [Back/](Back/README.md) y de [Front/](Front/README.md), asi como [Back/docs/a2ui-contract.md](Back/docs/a2ui-contract.md) (contrato entre agente y frontend) y [Back/docs/api-reference.md](Back/docs/api-reference.md) (referencia de endpoints).
