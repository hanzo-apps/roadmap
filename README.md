# Trailmap — Public Product Roadmap

<!-- hanzo-badges -->
<p align="left">
  <a href="https://platform.hanzo.ai/new?repo=https%3A%2F%2Fgithub.com%2Fhanzo-apps%2Froadmap&template=roadmap"><img src="https://gallery.hanzo.ai/deploy.svg" alt="Deploy to Hanzo" height="32" /></a>
  <a href="https://hanzo.ai/chat?template=roadmap"><img src="https://gallery.hanzo.ai/badges/edit-in-hanzo.svg" alt="Edit in Hanzo.ai" height="32" /></a>
  <a href="https://gallery.hanzo.ai/templates/roadmap"><img src="https://gallery.hanzo.ai/badges/live-preview.svg" alt="Live Preview" height="32" /></a>
  <a href="https://gallery.hanzo.ai/llms/roadmap/LLM.md"><img src="https://gallery.hanzo.ai/badges/llm-md.svg" alt="LLM.md Playbook" height="32" /></a>
</p>
<!-- /hanzo-badges -->

Show where you're headed. A public product roadmap organised as **Now / Next /
Later**: the team places initiatives into horizon lanes with a theme and a
status, and visitors follow what's in progress and what's coming.

A real, buildable Hanzo app you fork on [hanzo.app](https://hanzo.app) and deploy
live on Hanzo Cloud.

- **UI** — [`@hanzo/gui`](https://www.npmjs.com/package/@hanzo/gui) (the Hanzo
  design system) under Vite + React 19. 100% gui primitives — no Tailwind, no
  second kit.
- **Auth** — [`@hanzo/iam`](https://www.npmjs.com/package/@hanzo/iam), OAuth2
  **PKCE** against [hanzo.id](https://hanzo.id). No local passwords — IAM owns
  every credential interaction.
- **Data** — [`@hanzo/base`](https://www.npmjs.com/package/@hanzo/base), the
  IAM-native, org-scoped data plane. `initiatives` and `themes` are real Base
  collections provisioned from [`schema.sql`](./schema.sql).

## The app

| View | What it does |
| --- | --- |
| **Roadmap** | Three horizon lanes (mint / sky / lilac) of initiative cards — each carrying its theme colour-bar and a status progress-dot. |
| **Initiative detail** | The goal (summary + notes), where it sits, and its status, with an Edit action. |
| **Editor** | Create or **move** an initiative — the horizon control is the move. Pick horizon, status, and a theme. |

The signed-out landing is a **public** preview (a sample roadmap) so visitors see
the shape of the board before signing in; the real board loads from Base once you
sign in.

## Stack (pinned)

| Package | Version |
| --- | --- |
| `react` / `react-dom` | `^19.2.4` |
| `@hanzo/gui` + `@hanzogui/config` | `7.3.0` |
| `@hanzo/iam` | `^0.13.1` |
| `@hanzo/base` | `^0.2.1` |
| `vite` | `^6` (`@vitejs/plugin-react`) |
| `react-native-web` | `^0.21.0` |
| `typescript` | `5.9.3` |

## Run it

```sh
npm install
npm run dev        # http://localhost:5173
npm run build      # tsc --noEmit && vite build  ->  dist/
npm run preview    # serve the production build (SPA fallback on)
```

Out of the box it runs against **live** Hanzo (hanzo.id + api.hanzo.ai) — no
config needed to see the landing and the sign-in flow. Copy `.env.example` to
`.env` to point at a different environment.

## Environment contract

Only `VITE_`-prefixed vars reach the browser (this is a static SPA — there is no
server). Defaults in parentheses.

| Var | Purpose |
| --- | --- |
| `VITE_HANZO_IAM_URL` (`https://hanzo.id`) | OIDC issuer. |
| `VITE_IAM_CLIENT_ID` (`hanzo-app`) | IAM application (`<org>-<app>`). A per-app `hanzo-trailmap` client is provisioned at deploy; until then the shared `hanzo-app` client is the fallback. Its redirect-URI list must allow this deploy's `/auth/callback` — see **Ambient IAM**. |
| `VITE_HANZO_REDIRECT_URI` (`${origin}/auth/callback`) | PKCE redirect. |
| `VITE_HANZO_BASE_URL` (`https://api.hanzo.ai`) | Browser-reachable Hanzo Base data plane. Deploy injects the provisioned URL. |

## How auth works — ambient IAM

`login()` starts an OAuth2 **PKCE S256** redirect to hanzo.id; hanzo.id returns
to `/auth/callback`, where `handleCallback()` exchanges the code for tokens
(stored in `localStorage`, refresh-aware via `offline_access`). Every deployed
app is a static site at `<slug>.hanzo.app`; there is **no server token** — the
SPA authenticates the user in the browser and carries the resulting IAM JWT to
Base. The IAM client id is read from `VITE_IAM_CLIENT_ID` (fallback `hanzo-app`).

The one deploy requirement: that IAM client must list this origin's
`/auth/callback` as an allowed redirect URI — a `https://*.hanzo.app/auth/callback`
wildcard on the shared client, or a dedicated `hanzo-trailmap` client.

## How data works — Base from `schema.sql`

[`schema.sql`](./schema.sql) is the app's `databaseSchema` (SQL DDL). On publish,
Hanzo Cloud translates each `CREATE TABLE` into a Hanzo Base collection
(`provisionBaseFromDDL`, additive + idempotent). Base manages
`id`/`created`/`updated`/`owner`/`org`, stamps `owner`+`org` from the verified
IAM principal, and scopes every row to the caller's org (the isolation rule
`@request.auth.org_id = org` is enforced by Base) — a teammate in your org sees
the row; other orgs cannot. At runtime `src/views/roadmap.tsx` and `editor.tsx`
read/write the `initiatives` and `themes` collections through `@hanzo/base/react`
(`useQuery`/`useMutation`) carrying the IAM token. Keep `schema.sql` in lockstep
with `src/lib/model.ts`.

## Deploy — Hanzo Cloud

[`hanzo.yml`](./hanzo.yml) declares a static build (`npm run build` → `dist/`,
served at `<slug>.hanzo.app`) plus the Base schema to provision and the env to
inject. Do **not** build a container image locally — Hanzo Cloud owns builds and
deploys. CI here only proves the template compiles green.

## Layout

```
src/
  main.tsx          entry
  providers.tsx     GuiProvider(light) -> IamProvider -> BaseProvider(client=IAM-token)
  app.tsx           route (/auth/callback) + auth gate
  gui.config.ts     createGui(defaultConfig from @hanzogui/config/v5)
  iam.config.ts     IAM PKCE config (appName product-trailmap)
  env.ts            the VITE_ env contract, one place
  lib/base.ts       BaseClient carrying the IAM bearer token
  lib/model.ts      horizons, statuses, theme palette — the roadmap's vocabulary
  auth/callback.tsx PKCE return leg
  views/
    signed-out.tsx  public landing + sample roadmap preview
    roadmap.tsx     the Now / Next / Later board (signed-in shell)
    initiative.tsx  initiative detail
    editor.tsx      create / move initiative
    card.tsx        one initiative card (shared by board + landing)
    brand.tsx       the Trailmap wordmark
schema.sql          databaseSchema -> Base collections on publish
hanzo.yml           Hanzo Cloud build/deploy manifest
```
