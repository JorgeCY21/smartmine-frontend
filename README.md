<div align="center">

# ⛏️ SmartMine AI — Frontend

**Dashboard de simulación y diseño de operaciones mineras en tiempo real**

[![React](https://img.shields.io/badge/React-19.2-61DAFB?logo=react&logoColor=white)](https://react.dev/)
[![Vite](https://img.shields.io/badge/Vite-8.2-646CFF?logo=vite&logoColor=white)](https://vitejs.dev/)
[![Tailwind CSS](https://img.shields.io/badge/Tailwind_CSS-3.4-38B2AC?logo=tailwindcss&logoColor=white)](https://tailwindcss.com/)
[![Konva](https://img.shields.io/badge/Canvas-Konva-0D83CD)](https://konvajs.org/)
[![WebSocket](https://img.shields.io/badge/Realtime-WebSocket-4FC08D)](#persistencia-y-tiempo-real)

</div>

---

## Tabla de contenidos

1. [Descripción general](#descripción-general)
2. [Stack tecnológico](#stack-tecnológico)
3. [Arquitectura](#arquitectura)
4. [Estructura del repositorio](#estructura-del-repositorio)
5. [Modelo de datos](#modelo-de-datos)
6. [API y comunicación en tiempo real](#api-y-comunicación-en-tiempo-real)
7. [Persistencia y tiempo real](#persistencia-y-tiempo-real)
8. [Puesta en marcha](#puesta-en-marcha)
9. [Scripts disponibles](#scripts-disponibles)
10. [Variables de entorno](#variables-de-entorno)
11. [Servicios y dependencias externas](#servicios-y-dependencias-externas)
12. [Calidad de código, testing y CI/CD](#calidad-de-código-testing-y-cicd)
13. [Limitaciones conocidas y deuda técnica](#limitaciones-conocidas-y-deuda-técnica)
14. [Roadmap sugerido](#roadmap-sugerido)
15. [Licencia](#licencia)

---

## Descripción general

**SmartMine AI** (nombre visible en la UI, `src/pages/ModuleSelector.jsx`; el `package.json` conserva el nombre genérico `"dashboard"` heredado de la plantilla) es un **dashboard de simulación de logística minera a cielo abierto**: modela el acarreo de camiones entre palas de carga y una estación de descarga sobre un grafo de nodos y rutas, y compara el rendimiento de una asignación optimizada ("SmartMine") frente a una asignación tradicional/baseline.

La aplicación ofrece dos módulos, seleccionables desde la pantalla de inicio ([`src/pages/ModuleSelector.jsx`](src/pages/ModuleSelector.jsx)):

- **Simulación** ([`src/modules/simulation/`](src/modules/simulation/)): observa en tiempo real una flota preconfigurada operando sobre un mapa de mina, con métricas en vivo (tiempo de espera promedio, ciclos completados, utilización, tiempo ahorrado) y comparación SmartMine vs. baseline.
- **Constructor** ([`src/modules/builder/`](src/modules/builder/)): permite diseñar un layout de mina desde cero (nodos, rutas, palas, estación de descarga, flota de camiones), validar su conectividad, lanzarlo como una sesión de simulación propia e inyectar eventos/fallas en vivo (caída de rutas, deshabilitar palas).

Es un **frontend puro**: toda la lógica de simulación (motor de asignación, física de movimiento de camiones, cálculo de métricas) vive en el backend hermano **`smartmine-backend`** (FastAPI), consumido vía HTTP + WebSocket. Este repositorio solo contiene la interfaz de visualización y control.

> El `README.md` original de este repositorio era el boilerplate por defecto de Vite ("React + TypeScript + Vite" / plantilla mínima) sin adaptar al proyecto real; este documento lo reemplaza con documentación derivada de una lectura directa del código fuente.

## Stack tecnológico

| Categoría | Tecnología | Versión |
|---|---|---|
| Librería UI | React / React DOM | `^19.2.8` |
| Bundler / dev server | Vite | `^8.2.0` (plugin `@vitejs/plugin-react` `^6.0.4`) |
| Lenguaje | JavaScript (JSX) — **sin TypeScript real** | — |
| Enrutamiento | React Router DOM | `^7.18.2` |
| Render de mapa/canvas | Konva + react-konva | `^10.3.0` / `^19.2.5` |
| Estilos | Tailwind CSS + PostCSS + Autoprefixer | `^3.4.19` |
| Linting | oxlint (linter en Rust, alternativa a ESLint) | `^1.75.0` |
| Gestor de paquetes | npm (`package-lock.json`) | — |
| Comunicación en tiempo real | WebSocket nativo del navegador (sin socket.io) | — |

**Notas sobre el stack:**
- `@types/react` y `@types/react-dom` están instalados como devDependencies, pero **no hay `tsconfig.json` ni un solo archivo `.ts`/`.tsx`** en el proyecto — son dependencias sin uso real (probablemente solo para autocompletado del editor) o residuo de una migración a TypeScript que no se completó.
- No hay librería de gráficas tradicional (Chart.js, Recharts, D3): las métricas se muestran como tarjetas de texto ([`src/components/MetricCard.jsx`](src/components/MetricCard.jsx)); el "mapa" es un canvas 2D dibujado con Konva, no un mapa geoespacial real.
- No hay gestor de estado global (Redux/Zustand/Context): el estado se maneja con hooks locales y un hook compartido de datos en vivo ([`src/useSimState.js`](src/useSimState.js)).
- No hay librería de formularios; los inputs (distancias de ruta, tipo/nodo inicial de camión) son controlados manualmente con `useState`.
- No hay `engines` en `package.json` ni `.nvmrc`; Vite 8 requiere Node moderno (≥18/20).

## Arquitectura

Frontend SPA que consume un backend externo (proyecto hermano `smartmine-backend`, FastAPI) vía **HTTP para comandos** y **WebSocket para datos en vivo**:

```
┌───────────────────────────────────────────────────────────────┐
│                    smartmine-frontend (SPA)                    │
│                                                                 │
│  react-router-dom ──▶ pages/ModuleSelector                     │
│                              │                                 │
│              ┌───────────────┴────────────────┐                │
│              ▼                                 ▼                │
│   modules/simulation/               modules/builder/            │
│   (flota preconfigurada,            (diseño de mina,             │
│    sesión "default")                 sesión "custom")            │
│              │                                 │                │
│              └───────────────┬─────────────────┘                │
│                               ▼                                 │
│                     src/useSimState.js                          │
│         (fetch de comandos + WebSocket de estado en vivo)       │
└───────────────────────────────────────────────────────────────┘
                                │
                     HTTP (POST) + WS (/ws?session=)
                                │
                                ▼
                  smartmine-backend (FastAPI, repo hermano)
             motor de simulación, asignación óptima, métricas
```

Decisiones de arquitectura relevantes (documentadas inline en el propio código, no en un README aparte):

- **WebSocket en vez de polling**: comentario explícito en [`src/useSimState.js:39-41`](src/useSimState.js) — *"Reemplaza el polling REST (3 peticiones/seg por vista) por un único WebSocket por sesión que recibe trucks/shovels/metricas/graph en cada tick."* El hook `useLiveSession` reconecta automáticamente cada 1s si el socket se cae.
- **Sesiones múltiples**: el backend soporta sesiones simultáneas identificadas por query param (`?session=default` para el módulo de Simulación, `?session=custom` para el diseño del usuario en el Constructor) — no son sesiones de usuario/autenticación, sino instancias independientes del motor de simulación.
- **Suavizado de animación en cliente**: dado que el backend emite ticks a ~1/segundo, [`src/modules/simulation/SimCanvas.jsx`](src/modules/simulation/SimCanvas.jsx) implementa una doble capa de interpolación (progreso a lo largo de la ruta + posición x/y final) para animar los camiones de forma fluida entre ticks, evitando que se vean "teletransportados".
- **Organización por feature**: `src/modules/simulation/` y `src/modules/builder/` son subárboles independientes con su propio canvas, controles y hook de estado (`useBuilderState.js`); `src/components/` agrupa piezas transversales reutilizadas por ambos (leyenda de estados, tabla de colas de palas, panel de comparación, panel de avisos).
- **Sin autenticación**: no hay login, tokens ni control de sesión de usuario — cualquier cliente con acceso a `VITE_API_BASE` puede iniciar/detener simulaciones o mutar el grafo (ver [Limitaciones](#limitaciones-conocidas-y-deuda-técnica)).

## Estructura del repositorio

```
smartmine-frontend/
├── public/
│   ├── camion_carga.png, camion_sin_carga.png, smartmine.png   # imágenes de marca/hero
│   ├── icons.svg, favicon.svg
│   └── mine_graph.json        # grafo de mina de ejemplo (fixture; sin import detectado en src/)
├── vite.config.js              # puerto configurable vía env var PORT
├── tailwind.config.js / postcss.config.js
├── .oxlintrc.json               # config de oxlint (reglas react/rules-of-hooks, etc.)
├── vercel.json                  # rewrite SPA para despliegue en Vercel
├── .env.example                 # única variable: VITE_API_BASE
├── index.html
├── package.json
│
└── src/
    ├── main.jsx                 # entry point (createRoot)
    ├── App.jsx                  # definición de rutas (react-router-dom)
    ├── index.css                # directivas Tailwind
    ├── useSimState.js           # capa de acceso a la API del backend (HTTP + WebSocket)
    │
    ├── pages/
    │   └── ModuleSelector.jsx   # landing: selección entre Simulación y Constructor
    │
    ├── modules/
    │   ├── simulation/
    │   │   ├── SimulationView.jsx   # vista principal del módulo de simulación
    │   │   ├── SimCanvas.jsx        # render Konva + interpolación de movimiento
    │   │   └── Controls.jsx         # start/stop/reset/velocidad
    │   └── builder/
    │       ├── BuilderView.jsx      # vista principal del constructor
    │       ├── BuilderCanvas.jsx    # edición visual de nodos/rutas/palas
    │       ├── Toolbar.jsx          # herramientas de edición
    │       ├── EventPanel.jsx       # inyección de eventos/fallas en vivo
    │       └── useBuilderState.js   # estado del diseño + validación de conectividad + payload de creación
    │
    ├── components/                 # piezas compartidas entre módulos
    │   ├── AvisosPanel.jsx, ComparacionPanel.jsx, EstadoLegend.jsx
    │   ├── Logo.jsx, MetricCard.jsx, ShovelQueueTable.jsx, SpeedControl.jsx
    │
    ├── lib/                         # utilidades puras sin estado de React
    │   ├── bezier.js                 # geometría de curvas para rutas
    │   ├── theme.js                  # colores y etiquetas por estado de camión
    │   └── useImage.js               # hook para cargar imágenes en Konva
    │
    └── assets/                      # SVG/PNG de iconografía (truck, shovel, station, hero)
```

## Modelo de datos

No hay base de datos ni ORM en el frontend: consume el estado del backend tal cual, sin capa de validación de forma (no hay TypeScript, PropTypes ni Zod). Las entidades, tal como se usan en el código:

| Entidad | Campos principales | Uso |
|---|---|---|
| **Nodo** | `id, x, y` (+ `tipo`: `nodo\|pala\|estacion` en el constructor) | Punto del grafo de la mina |
| **Arista/ruta** | `from, to, distancia_km, cp` (`cp` = punto de control opcional para curva bezier) | Conexión entre nodos |
| **Camión** | `id, tipo, nodo_inicial, estado, path, progreso_km, x, y, _fase` | Unidad de acarreo; estados en [`src/lib/theme.js`](src/lib/theme.js): `idle`, `hauling_empty`, `hauling_loaded`, `queued`, `loading`, `dumping` |
| **Pala (shovel)** | `id, nodo_id, capacidad_cola, cola_actual, enabled` | Punto de carga con cola de camiones |
| **Métricas** | `tiempo_espera_promedio, ciclos_completados, tiempo_ciclo_promedio, camiones_activos, tiempo_ahorrado_min, modo, utilizacion_pct, comparacion:{smartmine, baseline}, speed_multiplier` | Panel de indicadores en vivo |
| **Grafo** | `nodos, aristas, entrada` (`entrada` = nodo de descarga) | Layout completo de la mina |

Payload de creación de una sesión personalizada (`createCustomSession`, [`src/useSimState.js`](src/useSimState.js)): `{ nodes, edges, trucks, shovels, dump_node }`, construido por `toPayload()` en [`src/modules/builder/useBuilderState.js`](src/modules/builder/useBuilderState.js).

> `public/mine_graph.json` parece un fixture/dataset de ejemplo (nodos con coordenadas), pero no se detectó ningún `import` o `fetch` hacia ese archivo en `src/` — podría ser un artefacto de referencia sin uso real en tiempo de ejecución.

## API y comunicación en tiempo real

Toda la comunicación con el backend pasa por [`src/useSimState.js`](src/useSimState.js), con base en `VITE_API_BASE` (HTTP) y su equivalente WebSocket (`ws://`/`wss://`, derivado automáticamente reemplazando el esquema `http`):

| Endpoint | Método | Función | Descripción |
|---|---|---|---|
| `/ws?session=` | `WS` | `useLiveSession(session, enabled)` | Stream en vivo de `trucks`, `shovels`, `metricas`, `graph`, `avisos` |
| `/sim/start?session=` | `POST` | `startSim` | Inicia la simulación de una sesión |
| `/sim/stop?session=` | `POST` | `stopSim` | Detiene la simulación |
| `/sim/reset?session=` | `POST` | `resetSim` | Reinicia el estado de la sesión |
| `/speed/{multiplier}?session=` | `POST` | `setSpeed` | Ajusta el multiplicador de velocidad de la simulación |
| `/graph/road/toggle?session=` | `POST` | `toggleRoad(from, to)` | Habilita/deshabilita una ruta entre dos nodos |
| `/graph/shovel/{id}/toggle?session=` | `POST` | `toggleShovel(id)` | Habilita/deshabilita una pala |
| `/trucks/add?session=` | `POST` | `addTruck(tipo, nodoInicial)` | Agrega un camión a la flota en vivo |
| `/trucks/{id}/remove?session=` | `POST` | `removeTruck(id)` | Retira un camión de la simulación |
| `/custom/create` | `POST` | `createCustomSession(payload)` | Crea la sesión `custom` a partir del diseño del Constructor |

El backend (`smartmine-backend`) expone además `/state`, `/metrics`, `/graph` (GET) y `/sim/speed/{multiplier}`, que **este frontend no consume** — posibles endpoints legacy o duplicados del lado del backend (el frontend usa `/speed/{multiplier}`, no `/sim/speed/{multiplier}`). No existe una especificación OpenAPI/Swagger versionada; el contrato de la API vive únicamente como código en el backend.

## Persistencia y tiempo real

- No hay persistencia del lado del cliente (sin `localStorage`/`sessionStorage`, sin base de datos local): todo el estado vivo proviene del WebSocket del backend.
- El backend es la única fuente de verdad. El frontend no cachea ni reconstruye estado entre recargas de página — al refrescar, se reconecta al WebSocket y recibe el estado actual de la sesión.
- Reconexión automática del WebSocket cada 1 segundo ante desconexión ([`src/useSimState.js:65-67`](src/useSimState.js)).
- `TRUCK_SPEED_KMH = 30` está hardcodeado en [`src/modules/simulation/SimCanvas.jsx`](src/modules/simulation/SimCanvas.jsx) con un comentario indicando que debe coincidir manualmente con la constante equivalente en el backend (`simulator.py`) — acoplamiento frágil sin fuente única de verdad entre ambos repositorios.

## Puesta en marcha

**Requisitos**: Node.js ≥18 (recomendado 20+), npm, y una instancia de `smartmine-backend` corriendo (por defecto en `http://localhost:8000`).

```bash
# Clonar el repositorio
git clone <url-del-repositorio>
cd smartmine-frontend

# Instalar dependencias
npm install

# Configurar variables de entorno
cp .env.example .env
# editar .env si el backend no corre en http://localhost:8000

# Levantar el servidor de desarrollo (Vite, con HMR)
npm run dev
```

La app queda disponible en `http://localhost:5173` (o el puerto indicado por la variable de entorno `PORT`, ver [`vite.config.js`](vite.config.js)). Debe estar corriendo `smartmine-backend` para que la app funcione, ya que no hay datos ni modo offline/mock.

## Scripts disponibles

Definidos en [`package.json`](package.json):

| Script | Comando | Descripción |
|---|---|---|
| `npm run dev` | `vite` | Servidor de desarrollo con HMR |
| `npm run build` | `vite build` | Build de producción en `dist/` (sin verificación de tipos, ya que no hay TypeScript real) |
| `npm run lint` | `oxlint` | Linting rápido con oxlint (reglas `react/rules-of-hooks`, `react/only-export-components`, definidas en [`.oxlintrc.json`](.oxlintrc.json)) |
| `npm run preview` | `vite preview` | Sirve localmente el build de producción |

No existe script `test` ni `typecheck`.

## Variables de entorno

Definidas en [`.env.example`](.env.example):

| Variable | Descripción | Valor de ejemplo |
|---|---|---|
| `VITE_API_BASE` | URL base HTTP del backend `smartmine-backend`; se deriva automáticamente a WebSocket (`ws://`/`wss://`) reemplazando el esquema `http` | `http://localhost:8000` |

> ⚠️ Si `VITE_API_BASE` no está definida, [`src/useSimState.js:4`](src/useSimState.js) lanza un error en tiempo de ejecución al intentar hacer `.replace()` sobre `undefined` — no hay valor por defecto ni validación explícita. Asegurarse de copiar `.env.example` a `.env` antes de ejecutar `npm run dev`.

## Servicios y dependencias externas

- **`smartmine-backend`** (repositorio hermano, FastAPI): único servicio consumido, expuesto vía HTTP + WebSocket. No hay documentación OpenAPI/Swagger compartida entre ambos repos.
- No hay APIs de terceros: sin mapas geoespaciales reales (Google Maps/Mapbox), sin analytics, sin Sentry/LogRocket, sin pasarela de pago.
- No hay autenticación de terceros (OAuth, Firebase, Auth0): la aplicación no tiene concepto de usuario/login, solo "sesiones" de simulación (`default`/`custom`).

## Calidad de código, testing y CI/CD

- **Linting**: `oxlint` (linter en Rust, alternativa moderna a ESLint) con reglas de React ([`.oxlintrc.json`](.oxlintrc.json)). No hay Prettier configurado.
- **Testing**: **no hay ningún framework de testing instalado** (sin Vitest, Jest, React Testing Library), ni archivos `*.test.jsx`/`*.spec.jsx`. Esto es una brecha relevante dado que hay lógica no trivial sin cobertura: validación de conectividad de grafo en `useBuilderState.js`, e interpolación geométrica de movimiento en `SimCanvas.jsx`.
- **CI/CD**: no existe `.github/workflows/`, `Dockerfile` ni `sonar-project.properties`. El único artefacto de despliegue es [`vercel.json`](vercel.json) (rewrite SPA para Vercel); no hay pipeline que ejecute lint/build automáticamente antes de desplegar.

## Limitaciones conocidas y deuda técnica

- 📦 **`package.json` desalineado con el proyecto real**: `name: "dashboard"`, sin `description`, `version: "0.0.0"` — residuo de la plantilla inicial de Vite, nunca actualizado para reflejar el nombre real del producto ("SmartMine AI").
- 🟨 **TypeScript a medias**: `@types/react`/`@types/react-dom` instalados sin ningún archivo `.ts`/`.tsx` ni `tsconfig.json` en el proyecto — dependencias sin uso real, o una migración a TypeScript que nunca se completó.
- 🧪 **Cobertura de pruebas nula**: sin tests unitarios ni de integración, pese a lógica de negocio no trivial (validación de grafos, interpolación de movimiento).
- 🔓 **Sin autenticación ni control de acceso**: cualquier cliente con la URL del backend puede iniciar/detener simulaciones o mutar el grafo en vivo (`/trucks/add`, `/graph/road/toggle`, etc.) — aceptable para una demo/PoC, pero inseguro si se expone públicamente sin una capa adicional de control de acceso (el backend tampoco parece implementar autenticación).
- ⚠️ **`VITE_API_BASE` sin fallback**: si la variable de entorno no está definida, la app falla al iniciar el WebSocket en vez de mostrar un error controlado.
- 🔁 **Constantes duplicadas entre frontend y backend**: `TRUCK_SPEED_KMH = 30` está hardcodeado en ambos repositorios de forma independiente, con un comentario que exige mantenerlos sincronizados manualmente — riesgo de desincronización silenciosa.
- 🧹 **Posibles endpoints huérfanos en el backend**: `/state`, `/metrics`, `/graph` (GET) y `/sim/speed/{multiplier}` existen en `smartmine-backend` pero no los consume este frontend — revisar si son legacy y pueden eliminarse, o si faltan integrarse.
- 🗺️ **`public/mine_graph.json` sin referencia detectada**: posible archivo huérfano o de solo referencia, sin `import`/`fetch` desde `src/`.
- 📄 **Sin `LICENSE`** versionado en el repositorio.
- 📚 **Sin especificación de API compartida**: no hay contrato OpenAPI/Swagger entre frontend y backend; cualquier cambio de forma en las respuestas del WebSocket o los endpoints requiere actualizar ambos repos manualmente sin validación automática.

## Roadmap sugerido

- [ ] Actualizar `name`/`description`/`version` en `package.json` para reflejar el proyecto real.
- [ ] Decidir entre completar la migración a TypeScript (aprovechando los `@types/*` ya instalados) o eliminarlos si no se van a usar.
- [ ] Incorporar una suite de tests (Vitest + React Testing Library) para `useBuilderState.js` (validación de conectividad) y para la interpolación de movimiento en `SimCanvas.jsx`.
- [ ] Añadir un valor de fallback o validación explícita para `VITE_API_BASE`.
- [ ] Publicar una especificación OpenAPI del backend y generar/validar los tipos de datos del frontend contra ella.
- [ ] Extraer `TRUCK_SPEED_KMH` (y otras constantes compartidas) a un único origen de verdad, o documentarlas explícitamente como contrato entre ambos repos.
- [ ] Añadir un pipeline de CI (lint + build en cada PR).
- [ ] Definir si `smartmine-backend` requiere autenticación antes de cualquier despliegue público, y reflejarlo en el frontend.
- [ ] Añadir un archivo `LICENSE`.

## Licencia

No se incluye archivo `LICENSE` en el repositorio. Definir y agregar la licencia correspondiente antes de una distribución formal del proyecto.
