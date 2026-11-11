# Full-stack code and deployment guidelines

A reusable reference based on Katavti's TypeScript server/client architecture. Use this when starting another project or reviewing an existing one.

**Scope:** application boundaries, framework choices, server configuration, local development, and Kubernetes deployment. Examples use a fictional application named `example`, the environment prefix `APP_`, and API prefix `/api/v1`. Rename these consistently. These examples are recommended templates, not files already implemented in Katavti.

## 1. Architecture and framework choices

Use two independently buildable applications, exposed through one browser origin:

```text
Browser: https://app.example.com
                 |
              Ingress
           /           \
      /api/v1/*          /*
     API Service     Client Service
          |                |
     Hono / Node       static nginx
          |
  services + storage adapters
```

| Concern | Katavti implementation | Reusable guideline |
| --- | --- | --- |
| Server | Hono with `@hono/node-server` | Small route modules; standard Request/Response APIs; Node only at infrastructure boundaries. |
| Server language | Strict TypeScript, ESM, `NodeNext` | Compile server code; use `.js` extensions in relative server imports so emitted ESM runs directly. |
| Browser | Vue 3 single-file components | Views compose components; components render state and emit intent. |
| Browser tooling | Vite, `@vitejs/plugin-vue`, `vue-tsc` | Use Vite for development/builds and run type checking separately or before build. |
| Routing/state | Vue Router 4, Vuex 4 | Make navigation explicit; centralize shared workflows. Vuex is the existing choice, not a requirement for every new project. |
| Tests | Vitest, Vue Test Utils/jsdom, Playwright | Test business boundaries independently, then HTTP and browser integration. |
| Production runtime | Node API + nginx static client | Ship compiled artifacts; no Vite or TypeScript watcher in production. |

Keep lockfiles committed and use `npm ci` in CI and image builds. Pin and test a supported Node version across local tooling, CI, and containers. Katavti has separate root/server and client package manifests and lockfiles; it is not an npm-workspaces setup. New projects may use workspaces, but should not combine both dependency models accidentally.

### Suggested repository layout

```text
package.json                 # server dependencies and orchestration scripts
package-lock.json
config-defaults.yaml         # safe development settings only
server/
  src/
    server.ts                # executable startup and shutdown
    app.ts                   # HTTP composition, no listening socket
    config/
      index.ts               # source loading, merging, normalization
      schema.ts              # one authoritative runtime schema
    routes/<feature>/index.ts
    services/                # workflows, authentication, domain operations
    data-modules/            # persistence contracts + implementations
    session-modules/         # independently selected session backends
  test/unit/
  test/e2e/
  tsconfig.json
  Dockerfile
client/
  package.json
  package-lock.json
  src/
    main.ts
    api/                     # HTTP and browser persistence adapters
    store/                   # shared async workflows and application state
    router/
    views/
    components/
  tests/unit/
  vite.config.ts
  Dockerfile
  nginx.conf
kubernetes/
  base/
  overlays/dev/
  overlays/prod/
```

Do not add layers merely to fill directories. A route may call a persistence contract directly for simple CRUD. Introduce a service when the operation coordinates multiple dependencies or has domain rules.

## 2. Server boundaries and dependency injection

**Rules**

- `server.ts` loads configuration, initializes dependencies, starts the listener, and owns shutdown.
- `app.ts` assembles middleware and feature routers without opening a port.
- Routes translate HTTP inputs/outputs and authenticate callers; services implement workflows.
- Storage modules own filesystem/cloud details. Routes must not know about Azure SDKs or file paths.
- Derive identity from a verified session, not an arbitrary browser-supplied identity header.
- Pass narrow config sections and dependencies into factories. Do not read environment variables throughout services.
- Return a stable JSON error shape and avoid exposing internal stack traces or secrets.

Katavti's `createApp(config, injectedNotesData?, injectedAuth?)` already enables HTTP tests without a real network listener. Its `startServer(config)` initializes the configured session backend before listening. Prefer a named dependency object as the number of dependencies grows. Move note-backend selection out of `createApp` into startup: the HTTP application should receive initialized dependencies rather than optionally constructing infrastructure itself.

### The plugin abstraction pattern

Use application-specific plugins for databases, blob storage, session stores, message brokers, and other external data services. A plugin consists of:

1. A consumer-driven contract describing the operations the application needs.
2. An initialization function accepting explicit configuration and dependencies.
3. An implementation returning those operations while hiding connection management and vendor-specific details.
4. Optionally, and recommended, a function returning config validators for collection before application validation (see §3).

Here, **plugin** means an injected implementation, not necessarily dynamic loading, a plugin marketplace, or an extension framework. Katavti's `NotesData` and its filesystem/Azure implementations already follow this pattern. No general-purpose plugin framework is required.

Design methods from application workflows, not vendor APIs. An interface should give newcomers a quick inventory of the application's interactions with its data service:

```ts
interface Farmer { id: string; name: string }
interface NewFarmer { name: string }

interface FarmersData {
  listFarmers(): Promise<Farmer[]>
  getFarmer(id: string): Promise<Farmer | undefined>
  addFarmer(input: NewFarmer): Promise<{ id: string }>
}
```

Do not expose `getMongoClient()`, `getCollection()`, SQL query builders, or Azure container clients to routes and domain services. That relocates connection code without removing vendor coupling. Prefer domain operations over a universal CRUD abstraction; standard primitives such as streams are appropriate when they match the actual consumer need.

Startup is the **composition root**: it loads configuration, selects implementations, validates their settings, initializes plugins, and injects them into consumers. Backend-specific configuration remains visible in startup and plugin factories, not in routes. This makes production wiring discoverable to developers and operators.

```ts
// server/src/data-modules/index.ts
import type { NotesData } from './notes-data.js'
import { createFileNotesData } from './file-notes-data.js'
import { createAzureNotesData } from './azure-notes-data.js'

export type NotesConfig =
  | { module: 'filesystem'; dataDir: string }
  | {
      module: 'azure'
      azure: { accountName: string; containerName: string }
    }

export function initializeNotesData(config: NotesConfig): NotesData {
  switch (config.module) {
    case 'filesystem':
      return createFileNotesData({ dataDir: config.dataDir })
    case 'azure':
      return createAzureNotesData(config.azure)
  }
}
```

```ts
// server/src/server.ts — illustrative wiring after config validation
const config = await loadConfig({ defaultsFile })
const notes = initializeNotesData(config.storage)
const sessions = await createSessionManager(config.sessions)
const auth = createAuthService({
  ...config.auth,
  environmentType: config.environmentType,
  sessions,
})
const app = createApp(config, { notes, auth })
```

A shared infrastructure library may own connection setup, credential handling, and SDK utilities. Each application's plugin owns its application-specific operations. Share infrastructure without forcing every service to adopt one generic repository interface.

### Plugin semantics and lifecycle

Implementations are interchangeable only when they preserve **behavior**, not merely TypeScript signatures. Document missing-record behavior, expected errors, ordering/pagination, revision conflicts, atomicity, consistency, retry/idempotency behavior, ID opacity, and resource ownership. Contracts can evolve with application needs; update implementations and tests together.

For Katavti, filesystem and Azure plugins must both preserve stale-write rejection and collision-free imports. A configuration switch alone does not prove equivalence.

Plugins owning connections, background tasks, or subscriptions should expose cleanup. Startup owns initialization; shutdown owns cleanup, including cleanup of already-initialized plugins if a later initialization fails.

```ts
interface InitializedPlugin<T> {
  operations: T
  close(): Promise<void>
}
```

Use this wrapper only when lifecycle management is needed; simple factories do not need artificial lifecycle methods.

For streaming plugins, document backpressure, cancellation, stream errors, and publication completion. A writable stream accepting bytes does not necessarily mean a cloud object has been durably published:

```ts
import type { Writable } from 'node:stream'

interface Upload {
  id: string
  writable: Writable
  completed: Promise<void> // resolves after successful publication
  abort(): Promise<void>  // cancels and cleans up incomplete publication
}
```

The handler should await both byte transfer and publication before reporting success and handle failures from both. Specify whether an ID may be retrieved before completion. Storage and live-stream distribution may share stream-shaped APIs while having different delivery, replay, and lifetime guarantees; do not promise interchangeability without defining those semantics.

### Example: persistence contract and route composition

```ts
// server/src/data-modules/items-data.ts
export interface Item { id: string; title: string }
export interface ItemsData {
  list(userId: string): Promise<Item[]>
}
```

```ts
// server/src/routes/items/index.ts
import { Hono } from 'hono'
import type { ItemsData } from '../../data-modules/items-data.js'

export type HttpEnv = { Variables: { userId: string } }

export function itemRoutes(items: ItemsData) {
  const routes = new Hono<HttpEnv>()
  routes.get('/', async c => c.json(await items.list(c.get('userId'))))
  return routes
}
```

```ts
// server/src/app.ts
import { Hono } from 'hono'
import type { MiddlewareHandler } from 'hono'
import type { AppConfig } from './config/index.js'
import type { ItemsData } from './data-modules/items-data.js'
import { itemRoutes, type HttpEnv } from './routes/items/index.js'

interface Dependencies {
  items: ItemsData
  authenticate: MiddlewareHandler<HttpEnv> // verifies session and sets userId
}

export function createApp(config: AppConfig, deps: Dependencies) {
  const app = new Hono<HttpEnv>()
  const api = config.api.basePath

  app.get(`${api}/health`, c => c.json({ status: 'ok' }))
  app.use(`${api}/items`, deps.authenticate)
  app.use(`${api}/items/*`, deps.authenticate)
  app.route(`${api}/items`, itemRoutes(deps.items))
  app.notFound(c => c.json({ error: 'Not found' }, 404))
  app.onError((error, c) => {
    console.error(error) // replace with structured, redacted logging
    return c.json({ error: 'Internal server error' }, 500)
  })
  return app
}
```

Validate request bodies as `unknown` at the HTTP boundary. Runtime checks are necessary even when the browser uses TypeScript. Use domain-specific errors for expected failures such as missing records and revision conflicts, then map them to HTTP status codes centrally.

For editable resources, implement optimistic concurrency: read a revision, submit it with the next write, and atomically reject stale writes. Katavti uses filesystem locks and atomic publication, and Azure conditional operations, rather than relying only on browser-side save ordering.

## 3. Server configuration is a startup contract

### Source order

Adopt and document a deterministic, low-to-high precedence order:

1. Committed development defaults, **only in `dev`**.
2. Explicit regular YAML files, in listed order.
3. Explicit secret YAML files, in listed order.
4. Explicitly mapped application environment variables.
5. CLI `--section.key=value` overrides.

Recursively merge objects; replace arrays and scalar values. Resolve relative file selectors and data directories against a documented working directory. Load once, collect core and selected-plugin validators, run the combined application validation once, normalize once, then initialize dependencies and inject the resulting typed object.

Katavti's implementation is `server/src/config/index.ts`. It supports comma-separated file selectors and list overrides. `loadConfig` accepts `env`, `argv`, and `cwd`, allowing tests to avoid ambient process state. Its current loader merges and validates internally; supporting plugin-contributed validators requires exposing separate loading/validation phases or a validator-collection callback between them.

### Bootstrap settings must be decided before loading application values

Determine `environmentType` and file selectors from a small bootstrap contract before choosing whether defaults apply. For production, set the environment explicitly in the process environment or CLI:

```bash
APP_ENVIRONMENT_TYPE=prod \
  APP_CONFIG_FILES=/etc/example/config/production.yaml \
  APP_CONFIG_SECRET_FILES=/etc/example/secrets/auth.yaml \
  node dist/server/server.js
```

The loader in a new project must explicitly map these `APP_*` names; Katavti's existing loader uses `KATAVTI_*` instead.

**Katavti caveat:** its decision to exclude defaults is made before file contents are read. A YAML file that alone changes `environmentType` to `prod` does not reliably exclude development defaults. Set `KATAVTI_ENVIRONMENT_TYPE=prod` externally; in a new loader, reject disagreement between bootstrap mode and final mode. Defaults are still read to obtain selectors even in production, so the defaults file must currently exist and parse successfully.

### Example: complete non-secret production configuration

This shape follows Katavti's current configuration contract. A new application should remove properties it does not use.

```yaml
# production.yaml — mount from a ConfigMap
# APP_ENVIRONMENT_TYPE=prod must also be set in the Deployment.
environmentType: prod
server:
  host: 0.0.0.0
  port: 3030
storage:
  module: filesystem
  dataDir: /data/notes
sessions:
  module: filesystem
  lazy: true
  cache: true
  filesystem:
    directory: /data/sessions
http:
  cors:
    origins: [https://app.example.com]
    allowLoopbackInDevelopment: false
  identity:
    headerName: x-user-id # compatibility field; not authorization
api:
  basePath: /api/v1
auth:
  googleClientId: "" # public identifier, not a client secret
  magicLinkBaseUrl: https://app.example.com
  mailFrom: Example <no-reply@example.com>
config:
  files: []
  secretFiles: []
```

```yaml
# auth.yaml — inject via a Secret; never commit the real value
# Configure a real delivery provider or explicitly disable email sign-in.
auth:
  smtpUrl: smtps://USER:PASSWORD@smtp.example.com
```

For Azure notes storage, replace the `storage` section with:

```yaml
storage:
  module: azure
  azure:
    accountName: exampleaccount
    containerName: notes
    # Optional clientId for the selected user-assigned identity.
```

Katavti also supports an Azure connection string supplied through secrets. Prefer workload identity where available. An Azure VM managed identity setup does not automatically work inside a Kubernetes pod: configure the cluster's identity integration and storage permissions explicitly. Katavti currently selects `ManagedIdentityCredential` when `storage.azure.clientId` is supplied and `DefaultAzureCredential` otherwise. For federated Kubernetes workload identity, do not assume the former branch supports that credential flow; adapt credential selection or use the default chain with the cluster-provided identity environment. Azure notes storage does not replace the filesystem session backend.

### Optional plugin-provided configuration validators

**Recommended, but not mandatory:** each plugin module exports `getConfigValidators()` returning an array of config validators. This function belongs to the module/descriptor, not an initialized instance: it must be callable before plugin initialization. A plugin without this function is valid, but its required settings still need validation supplied by the application.

Use this startup sequence:

1. Resolve bootstrap mode and file selectors, then load and merge all config sources into an unvalidated candidate.
2. Select all required plugin modules from the candidate through an explicit trusted registry, including independently configured notes, sessions, messaging, and other dependencies.
3. Call `getConfigValidators()` on every selected plugin that provides it; append the returned validators to the core and cross-plugin validators.
4. Run the validation engine against the candidate with the complete validator list. Aggregate errors and stop on failure.
5. Normalize the validated candidate into the typed application configuration.
6. Initialize selected plugins with their narrow config sections, assemble the application, and start listening.

Selecting/importing a plugin is **not initializing it**. Module imports and validator factories must not connect to databases, fetch credentials, create directories, start background tasks, or open sockets. Validators should be deterministic and side-effect-free; connectivity checks belong to initialization/readiness. Keep credential values out of validation messages.

Because selection precedes full validation, selection code must handle missing/invalid selectors without crashing or silently choosing a backend. Use an allowlisted registry, never import arbitrary config-supplied paths. Bootstrap parsing/selection checks can record errors before the combined application validation; they must not initialize plugins. Core validators own selector enums; selected plugins own backend-specific requirements. Include cross-plugin constraints where necessary and do not require unused backends' settings.

```ts
// Shared validator contract, compatible with Katavti's validation engine.
export interface ConfigValidator {
  key: string // absolute path in the merged config
  validate(key: string, value: unknown): {
    level: 'valid' | 'warning' | 'error' | 'fatal'
    message?: string
  }
}
```

```ts
// server/src/data-modules/filesystem-plugin.ts
import type { ConfigValidator } from '../config/validation.js'
import { createFileNotesData } from './file-notes-data.js'

export function getConfigValidators(): ConfigValidator[] {
  return [{
    key: 'storage.dataDir',
    validate: (_key, value) =>
      typeof value === 'string' && value.trim() !== ''
        ? { level: 'valid' }
        : { level: 'fatal', message: 'must be a non-empty directory path' },
  }]
}

export const initialize = createFileNotesData
```

```ts
// Illustrative orchestration; loadCandidate/selectPlugins/normalizeConfig
// are new project helpers, not existing Katavti exports.
const candidate = await loadCandidate({ defaultsFile })
const selection = selectPlugins(candidate) // { plugins, issues }
const validators = [...coreValidators, ...crossPluginValidators]

for (const plugin of selection.plugins) {
  validators.push(...(plugin.getConfigValidators?.() ?? []))
}

const results = runConfigValidators(candidate, validators)
const issues = [
  ...selection.issues,
  ...results
    .filter(result => result.level === 'error' || result.level === 'fatal')
    .map(result => `${result.key}: ${result.message ?? 'invalid value'}`),
]
if (issues.length) throw new ConfigError(issues, results)

const config = normalizeConfig(candidate)
// Only now initialize plugins and start the application.
```

If plugins can be mounted under different config sections, accept a config-path prefix in the validator factory instead of hardcoding `storage`. Collect validators in a deterministic order, once per configured plugin binding. Multiple validators for the same key may intentionally check different constraints; do not discard them by deduplicating keys.

Validator arrays alone do not necessarily describe all allowed keys, conversions, or output types. Keep strict parsing, unknown-key rejection, and typed normalization consistent with the selected plugins' schemas. A descriptor may also expose schema metadata when needed; an optional validator function does not imply a mandatory plugin framework.

### Validation requirements

Maintain **one authoritative validation pipeline**, composed from core and plugin-owned schemas rather than competing copies of backend rules. Keep validators, types, environment mappings, and normalization consistent.

- Aggregate configuration errors and fail before accepting traffic.
- Validate enums, required fields, URL schemes, paths, port bounds, and conditional backend requirements.
- Parse booleans strictly: only `true` and `false`; reject `treu` rather than coercing it to `false`.
- Reject malformed YAML roots and unknown keys, including misspelled CLI options.
- Validate file-selector lists instead of silently dropping malformed values.
- Require production-safe auth behavior; reject or explicitly disable incomplete providers.
- Normalize optional fields so a claimed `AppConfig` really has its declared runtime shape.
- Reject invalid API prefixes, including ambiguous trailing slashes.
- Never log the entire configuration. Log only redacted safe fields or source names.

```ts
// Example strict parser for an environment/CLI boundary.
export function parseBoolean(key: string, value: string): boolean {
  if (value === 'true') return true
  if (value === 'false') return false
  throw new Error(`${key}: must be true or false`)
}

// Service configuration is explicit, not inferred from NODE_ENV.
interface EmailOptions {
  environmentType: 'dev' | 'prod'
  sendMail?: (message: string) => Promise<void>
}

export function assertEmailConfigured(options: EmailOptions) {
  if (options.environmentType === 'prod' && !options.sendMail) {
    throw new Error('Email sign-in is not configured')
  }
}
```

`NODE_ENV=production` is a runtime/library optimization setting. It is **not** a substitute for the application's validated environment mode. Gate fake login, development magic-link disclosure, and relaxed security behavior with the same validated application mode. The client hiding a development button is not a security boundary.

Configuration is a startup snapshot. Changing a mounted ConfigMap or Secret does not automatically reconfigure a running application. Trigger a rollout after changes, preferably with a pod-template checksum or generated, versioned config names.

## 4. Client boundaries and browser configuration

- `main.ts` wires Vue, router, and store.
- Views coordinate screen-level state; components own presentation and local interaction.
- Keep `fetch` and persistence logic in `src/api/`, not scattered across components.
- Inject adapters into store factories to test workflows without real network/localStorage.
- Treat local/offline and remote storage as implementations of a compatible contract.
- Preserve unsaved drafts on failures. Serialize/coalesce saves per resource; never blindly retry revision conflicts.
- Split a growing store by workflow or feature rather than putting every concern in one module.

### Example: same-origin API adapter

```ts
// client/src/api/http.ts
// Keep this path synchronized with server config, Ingress, and probes.
export const API_BASE_PATH = '/api/v1'

export class ApiError extends Error {
  constructor(message: string, readonly status: number) { super(message) }
}

export async function request<T>(path: string, init: RequestInit = {}): Promise<T> {
  const response = await fetch(`${API_BASE_PATH}${path}`, {
    ...init,
    credentials: 'include',
  })
  const body = await response.json()
  if (!response.ok) throw new ApiError(body.error ?? 'Request failed', response.status)
  return body as T // validate runtime response shape where needed
}
```

```ts
// client/src/api/items.ts
import { request } from './http'
export interface Item { id: string; title: string }
export const itemsApi = { list: () => request<Item[]>('/items') }
```

Use relative URLs by default. Service DNS names and internal ports are for pods, not browsers. One public origin simplifies cookies, CORS, OAuth origins, and deployment portability. If cross-origin calls are genuinely needed, configure a consistent base URL for **all** API adapters and exact credentialed CORS behavior; do not rely on a wildcard.

Katavti currently hardcodes `/api/v1beta` in both notes and auth adapters. Its notes adapter accepts `VITE_API_URL`, but auth does not. Changing only `api.basePath` on the server will break the client and existing probes/Ingress. In a new project, share the API prefix contract or test all consumers for agreement.

### Build-time versus runtime browser settings

`import.meta.env.VITE_*` values are compiled into browser assets. They are public, not secrets. Setting a container environment variable after the client image is built does not change the JavaScript.

```bash
docker build \
  -f client/Dockerfile \
  --build-arg VITE_GOOGLE_CLIENT_ID=public-client-id \
  -t registry.example.com/example-client:RELEASE \
  client
```

Katavti's client Dockerfile supports this build argument. If the same client image must work across environments with different public settings, implement a separate runtime public-config endpoint or generated file, load it before mounting Vue, and validate it. Do not expose the server's full config through that endpoint.

## 5. Development: same paths, different processes

### Plain local development

Run the API watcher and Vite concurrently. Vite proxies API traffic so the browser still uses relative URLs:

```ts
// client/vite.config.ts
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [vue()],
  server: {
    host: '0.0.0.0',
    port: 8080,
    strictPort: true,
    allowedHosts: ['example.local'],
    watch: { usePolling: true }, // use when mounted files need it
    proxy: { '/api': { target: 'http://127.0.0.1:3030' } },
  },
})
```

```json
{
  "scripts": {
    "dev": "concurrently \"npm run dev:server\" \"npm --prefix client run dev\"",
    "dev:server": "tsx watch server/src/server.ts",
    "build:server": "tsc -p server/tsconfig.json",
    "build": "npm run build:server && npm --prefix client run build",
    "start": "node dist/server/server.js"
  }
}
```

### Live development inside Kubernetes

Keep Services, Ingress, hostnames, and API paths the same as production. Replace only runtime images/commands and source/config mounts:

| Component | Development | Production |
| --- | --- | --- |
| API | Node image + source + `tsx watch` | immutable image + compiled Node entry point |
| Client | Node image + source + Vite HMR | immutable nginx image + static assets |
| Configuration | explicit `dev` + development defaults | explicit `prod` + complete config + secrets |
| Source | mounted or synchronized | absent |
| Data | isolated disposable or dev PVC | durable PVC/external backend + backups |
| Exposure | private workstation cluster | TLS and production security controls |

Use Kustomize overlays or equivalent explicit environment separation. Do not depend on applying competing Deployment YAML files in the correct order. Katavti currently applies `dev-deployments.yaml` over `base.yaml`; overlays are the recommended reusable improvement.

Example strategic-merge patch for the dev API, assuming the base Deployment is named `example-server`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: example-server
spec:
  replicas: 1
  template:
    spec:
      enableServiceLinks: false
      containers:
        - name: server
          image: node:22-bookworm-slim
          workingDir: /workspace/example
          command: [npm, run, dev:server]
          env:
            - name: APP_ENVIRONMENT_TYPE
              value: dev
            - name: APP_SERVER_HOST
              value: 0.0.0.0
            - name: APP_SERVER_PORT
              value: "3030"
            - name: APP_STORAGE_DATA_DIR
              value: /data/notes
            - name: APP_SESSIONS_FILESYSTEM_DIRECTORY
              value: /data/sessions
          volumeMounts:
            - name: source
              mountPath: /workspace/example
      volumes:
        - name: source
          hostPath:
            path: /absolute/path/on/kubernetes-node/example
            type: Directory
```

Patch the client analogously: `workingDir: /workspace/example/client`, `command: [npm, run, dev]`, Node image, and the same source volume. The base must supply named HTTP ports and readiness probes. Use a dev ConfigMap/Secret and dev data volume, not production credentials/data. If patching a production-style Deployment, remove inherited `NODE_ENV=production`, production config selectors/mounts, and incompatible security settings. A command/image patch alone is insufficient.

**Live-development constraints**

- `hostPath` refers to the **node's filesystem**, not necessarily the developer's host. It is suitable for a trusted single-node local cluster; use source synchronization or explicit node placement elsewhere.
- Katavti hardcodes one checkout path. Parameterize the dev overlay rather than copying that path.
- Host-installed `node_modules` must match the container's OS/architecture. A Debian dev image and an Alpine production image can differ for native packages. Prefer a development image built with dependencies or Linux-container-installed dependency volumes.
- Vite polling is configured by `server.watch.usePolling`; setting `WATCHPACK_POLLING` alone does not configure Vite.
- Verify API watcher behavior on the actual mount. Polling on the client does not prove that server source watching works.
- The Ingress must forward WebSocket upgrades for HMR. Let Vite infer the public browser host unless testing shows that explicit HMR configuration is needed.
- In separate pods, Vite's proxy target `localhost:3030` is not the API pod. Through Ingress, the API prefix goes directly to the API Service. If testing via a client-only port-forward, point the Vite proxy at the API Service instead.
- HMR validates browser module replacement; API source edits usually restart the server process. Neither should require rebuilding a production image.

```bash
kubectl apply -k kubernetes/overlays/dev
kubectl rollout status deployment/example-client
kubectl rollout status deployment/example-server
curl http://example.local/api/v1/health
```

For K3s, images built with Docker may need importing into K3s/containerd; other clusters require registry access or their own image-loading mechanism. This is a cluster concern, not application code. Confirm edits actually change responses/UI and that the browser HMR WebSocket connects.

## 6. Production: immutable artifacts and explicit runtime config

### Container builds

Use multi-stage builds, separate image build contexts, and a small runtime containing only production dependencies and emitted output:

```dockerfile
# server/Dockerfile — build context is repository root
FROM node:22-bookworm-slim AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY server ./server
RUN npm run build:server

FROM node:22-bookworm-slim AS runtime
WORKDIR /app
ENV NODE_ENV=production
COPY package.json package-lock.json ./
RUN npm ci --omit=dev && npm cache clean --force
COPY --from=build /app/dist ./dist
# Required by Katavti-style loaders even when defaults are not merged in prod.
COPY config-defaults.yaml ./config-defaults.yaml
USER node
EXPOSE 3030
CMD ["node", "dist/server/server.js"]
```

For production, pin base images and release images by tested version/digest; the tags above are illustrative. Maintain `.dockerignore` files excluding dependencies, local config, secrets, data, and generated artifacts. Bind the API to `0.0.0.0` inside containers. Prefer Node as the direct main process so termination signals reach it.

The client image should run `vue-tsc --noEmit` and `vite build`, then copy only `dist/` into nginx. An nginx listener on 8080 needs SPA fallback for Vue Router history URLs:

```nginx
server {
  listen 8080;
  server_name _;
  root /usr/share/nginx/html;

  location / {
    try_files $uri $uri/ /index.html;
  }
  location = /index.html {
    add_header Cache-Control "no-cache";
  }
  location /assets/ {
    try_files $uri =404;
    add_header Cache-Control "public, max-age=31536000, immutable";
  }
}
```

Long immutable cache lifetimes are appropriate for content-hashed assets, not `index.html` or runtime public config. The Ingress sends API requests to Node without rewriting the API prefix. nginx does not need to proxy API traffic in this topology.

### Kubernetes resource example

The following is a minimal production topology for a **single API replica with filesystem persistence**. Supply the referenced ConfigMap, Secret, images, TLS Secret, DNS, storage class, and Ingress controller. It is not a universal HA template.

Create non-secret configuration from the earlier YAML example:

```bash
kubectl create configmap example-config \
  --from-file=production.yaml=production.yaml \
  --dry-run=client -o yaml > kubernetes/example-config.yaml
```

Provision `example-auth` with the key `auth.yaml` through your secret manager or deployment system. Do not commit a rendered Secret containing credentials: base64 is not encryption. Keep the examples below in an explicit namespace in a real deployment, and use least-privilege access to secrets.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: example-data
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 10Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: example-server
spec:
  replicas: 1
  # Avoid overlapping writers/volume mounts with this filesystem topology.
  # Recreate causes deployment downtime; choose a different backend for HA.
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: example-server
  template:
    metadata:
      labels:
        app: example-server
    spec:
      enableServiceLinks: false
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 1000
      containers:
        - name: server
          image: registry.example.com/example-server:RELEASE
          ports:
            - name: http
              containerPort: 3030
          env:
            - name: NODE_ENV
              value: production
            - name: APP_ENVIRONMENT_TYPE
              value: prod
            - name: APP_CONFIG_FILES
              value: /etc/example/config/production.yaml
            - name: APP_CONFIG_SECRET_FILES
              value: /etc/example/secrets/auth.yaml
          resources:
            requests: {cpu: 100m, memory: 128Mi}
            limits: {cpu: "1", memory: 512Mi}
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop: [ALL]
          startupProbe:
            httpGet: {path: /api/v1/health, port: http}
            periodSeconds: 2
            failureThreshold: 30
          readinessProbe:
            httpGet: {path: /api/v1/health, port: http}
          livenessProbe:
            httpGet: {path: /api/v1/health, port: http}
          volumeMounts:
            - {name: config, mountPath: /etc/example/config, readOnly: true}
            - {name: secrets, mountPath: /etc/example/secrets, readOnly: true}
            - {name: data, mountPath: /data}
      volumes:
        - name: config
          configMap:
            name: example-config
        - name: secrets
          secret:
            secretName: example-auth
        - name: data
          persistentVolumeClaim:
            claimName: example-data
---
apiVersion: v1
kind: Service
metadata:
  name: example-server
spec:
  selector: {app: example-server}
  ports:
    - {name: http, port: 80, targetPort: http}
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: example-client
spec:
  replicas: 2
  selector:
    matchLabels: {app: example-client}
  template:
    metadata:
      labels: {app: example-client}
    spec:
      enableServiceLinks: false
      containers:
        - name: client
          image: registry.example.com/example-client:RELEASE
          ports:
            - {name: http, containerPort: 8080}
          resources:
            requests: {cpu: 50m, memory: 32Mi}
            limits: {cpu: 500m, memory: 128Mi}
          readinessProbe:
            httpGet: {path: /, port: http}
---
apiVersion: v1
kind: Service
metadata:
  name: example-client
spec:
  selector: {app: example-client}
  ports:
    - {name: http, port: 80, targetPort: http}
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example
spec:
  ingressClassName: traefik # replace with your installed controller's class
  tls:
    - hosts: [app.example.com]
      secretName: example-tls
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /api/v1
            pathType: Prefix
            backend:
              service:
                name: example-server
                port: {name: http}
          - path: /
            pathType: Prefix
            backend:
              service:
                name: example-client
                port: {name: http}
```

Set `enableServiceLinks: false` to prevent Kubernetes legacy Service environment variables from colliding with application configuration names. Katavti explicitly overrides `KATAVTI_SERVER_PORT` because its Service name generates a variable with that name; disabling service links removes that accidental coupling.

The API security context above matches the example Node image's UID. The client needs an nginx image/config designed for non-root operation before applying the same hardening. Add a read-only root filesystem and writable scratch mounts only after verifying each runtime's needs. Set resource values from measurements, not solely from these sample numbers.

### Persistence and scaling are architecture decisions

- `emptyDir` survives container restarts in the same pod, **not pod replacement**. Never use it for production notes or sessions.
- A PVC is persistence, not a backup. Define retention, recovery objectives, backup schedules, and restore tests.
- Persist sessions separately from note-backend selection, even when both directories share one PVC.
- Filesystem note locks depend on actual filesystem semantics; shared storage is not automatically safe merely because it supports multiple mounts.
- Katavti's pending magic-link tokens are process-local. Restarting the API invalidates outstanding links; multiple replicas can route verification to a different process.
- Cached sessions are process-local. A logout in one replica need not invalidate another replica's cached copy. Session records also need a server-enforced expiry policy; cookie expiry alone is insufficient.
- For multiple API replicas, implement shared token/session storage with expiry and consistent revocation, plus safe shared note storage. Azure notes alone do not make the whole application horizontally scalable.
- Do not treat sticky sessions as a replacement for durable, correctly invalidated authentication state.
- Backend selection is configuration, but backend replacement is an operational change. Review migration, credentials, permissions, persistence, consistency, and recovery before switching plugins. The abstraction isolates code changes; it does not migrate data or make infrastructure equivalent.
- Different plugins in development and production are valid, but CI must also exercise the production implementation and its behavioral contract.

### Operational requirements

Implement graceful SIGTERM/SIGINT shutdown: stop accepting new requests, drain active requests, close dependencies, and exit within `terminationGracePeriodSeconds`. Katavti currently does not install explicit shutdown handlers.

Separate liveness from readiness when dependencies matter. Liveness should detect an unhealthy process, not restart healthy pods during a cloud-storage outage. Readiness may check initialization and required dependencies. Katavti's health route is a simple response, not proof that data storage or email delivery is available.

Add structured/redacted logs, request IDs, latency/error metrics, TLS renewal, network restrictions, secret rotation, authentication rate limits, and appropriate CSRF/origin protections. Do not publish Vite, fake login, development links, source mounts, or development secrets on production endpoints.

## 7. Test the contracts, not just components

Before accepting a change:

- Server units: services, storage behavior, sessions, and configuration.
- Plugin contracts: run the same behavioral suite against every implementation, covering errors, conflicts, atomicity, ordering, and lifecycle/stream semantics where applicable. Supplement with backend-specific integration tests; mocks alone cannot verify storage guarantees.
- Plugin config validation: selected plugins contribute validators before the engine runs; unselected backends are not required; optional hooks may be absent; invalid selectors and combined core/plugin failures are reported; no initialization occurs before validation succeeds.
- HTTP integration: authentication, validation, errors, configured API prefix, and injected adapters via `app.request`.
- Real-server integration: ephemeral port, temporary filesystem, explicit dependency cleanup.
- Client units: adapters, store workflows, and user-visible Vue component behavior.
- Browser integration: cookies, auth, editor persistence, offline behavior, and cross-device conflicts.
- Deployment contracts: Ingress paths match client URLs, Service ports match container ports, and probes match the configured API prefix.
- Cluster smoke checks: TLS, history-route refreshes, API responses, pod replacement persistence, config failure, and dev WebSocket/HMR behavior.

### Example: test precedence without ambient state

```ts
import { expect, it } from 'vitest'
import { loadConfig } from '../../src/config/index.js'

it('CLI overrides environment and YAML', async () => {
  // Fixture files must contain all required settings for the selected mode.
  const config = await loadConfig({
    defaultsFile: '/tmp/test-fixture/config-defaults.yaml',
    cwd: '/tmp/test-fixture',
    env: { APP_SERVER_PORT: '4040' },
    argv: ['--server.port=5050'],
  })
  expect(config.server.port).toBe(5050)
})
```

Also test malformed booleans, unknown keys, missing files, YAML-only environment changes, production default exclusion, conditional backend requirements, and absence of development authentication routes in production. Use temporary fixtures in real tests, not fixed shared `/tmp` directories.

```bash
npm ci
npm --prefix client ci
npm test
npm run build
kubectl apply --dry-run=server -k kubernetes/overlays/prod
kubectl apply -k kubernetes/overlays/prod
kubectl rollout status deployment/example-server
kubectl rollout status deployment/example-client
curl --fail https://app.example.com/api/v1/health
curl --fail https://app.example.com/daily
```

Run deployment commands only against the intended context/namespace after review. A successful YAML parse or mocked manifest test does not prove an Ingress controller, volume, identity provider, or HMR connection works.

## 8. Katavti review: patterns to retain and gaps not to copy

This section records the inspected implementation, not a claim about a running cluster.

| Observation | Source | Recommendation |
| --- | --- | --- |
| App factory and startup are separate; routes accept adapters, but `createApp` can still select/construct note storage. | `server/src/app.ts`, `server/src/server.ts`, `server/src/routes/` | Retain named dependency injection; move plugin selection/initialization to startup and require initialized HTTP dependencies. |
| Note storage and session backends are independently selected. | `server/src/data-modules/`, `server/src/session-modules/` | Retain this separation; account for both in persistence/scaling. |
| Active validators are inline in the loader; another exported validator set is not used by it and has different rules. Plugin validator collection is not currently implemented. | `server/src/config/index.ts`, `server/src/config/validators.ts` | Consolidate into one pipeline; separate loading from validation so selected plugins can optionally contribute validators before validation and initialization. |
| Boolean env/CLI parsing maps every value other than case-insensitive `true` to false. Unknown CLI keys are accepted. Some auth fields are not fully validated/normalized. | `server/src/config/index.ts` | Strict parsing, unknown-key rejection, complete schema and runtime shape. |
| Production exclusion is selected before reading configured files. | `server/src/config/index.ts` | Explicit external mode selection and disagreement tests. |
| Fake login and secure cookies use application mode; development email-link disclosure still uses ambient `NODE_ENV`. | `server/src/routes/auth/index.ts`, `server/src/services/auth-service.ts` | Use validated mode consistently for all security-sensitive behavior. |
| Identity header config is validated but does not authenticate requests. | `server/src/app.ts`, `server/test/unit/app-config.test.ts` | Do not document it as trusted identity; remove unused config in new projects. |
| Server API prefix is configurable; client URLs, probes, and Ingress use fixed `/api/v1beta`. | `client/src/api/`, `kubernetes/` | Coordinate this contract and test it. |
| Base Kubernetes Deployment uses development defaults, HTTP-only local Ingress, `latest` images, and `emptyDir`. Docker `NODE_ENV=production` does not change application mode. | `kubernetes/base.yaml`, `kubernetes/ingress.yaml`, `server/Dockerfile` | Treat as local demonstration, not production-ready Kubernetes. |
| Dev Deployments mount a workstation-specific source path and host dependencies. | `kubernetes/dev-deployments.yaml`, `client/vite.config.ts` | Local-only, parameterized dev overlay; verify mounts, watching, dependencies, and HMR. |
| Existing production automation targets Docker Compose/Caddy with Ansible, not Kubernetes. | `ansible/README.md`, `ansible/roles/katavti/templates/` | Reuse explicit runtime config and immutable-image ideas, not an assumption that production K8s is already supplied. |
| Startup has no explicit signal-draining handlers; health does not check dependencies. | `server/src/server.ts`, `server/src/routes/health/index.ts` | Add shutdown and operational health contracts before production rollout. |

## References

- Tom Klinovsky, [Why you should use the plugin abstraction pattern](https://medium.com/@lutraman/why-you-should-use-the-plugin-abstraction-pattern-3c709548734d), October 25, 2022. Reviewed from the supplied PDF. The article informs the consumer-driven contracts, explicit initialization/configuration, and vendor-independent data/stream examples; lifecycle, behavioral tests, and optional plugin-validator collection are additional recommendations here.

**Bottom line:** retain a small Hono API, a Vue/Vite client, consumer-driven plugin contracts, deterministic startup configuration, and same-origin path routing. Startup collects core and optional selected-plugin validators before running application validation, then explicitly initializes and injects implementations; consumers remain independent of vendor APIs. Keep live-development tooling in a private dev environment; deploy immutable artifacts with explicit production configuration, durable state, and tested operational behavior.
