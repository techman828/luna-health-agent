# Luna Health Agent

A women's health tracking application with conversational AI assistance. The repository is named Luna Health Agent; the application UI and package metadata currently use **FemTracker**.

The project combines a Next.js frontend, Supabase authentication and database storage, and Python LangGraph agents connected through CopilotKit. It is a development project with working tracking interfaces and several incomplete integrations; see [Current limitations](#current-limitations) before relying on its behavior.

## Features

| Area | Route | Interface and functionality |
| --- | --- | --- |
| Dashboard | `/` | Health overview, quick records, tips, and navigation |
| Cycle tracking | `/cycle-tracker` | Period records, cycle history, and phase estimates |
| Symptoms and mood | `/symptom-mood` | Symptom severity, mood records, and trend views |
| Fertility | `/fertility` | Basal body temperature, cervical mucus, and ovulation test records |
| Nutrition | `/nutrition` | Meals, hydration, and nutrition focus tracking |
| Exercise | `/exercise` | Activity records, recommendations, and weekly progress |
| Lifestyle | `/lifestyle` | Sleep duration, sleep quality, and stress tracking |
| Health insights | `/insights` | Health scores, trends, correlations, and recommendations |
| Recipes | `/recipe` | AI recipe editing, dietary preferences, and saved recipes |
| Settings | `/settings` | Profile, preferences, notifications, and data management controls |

Most application pages are gated by the client-side authentication provider. Password recovery and callback pages live under `/auth`. A separate `/copilotkit` page contains the starter assistant demo.

## Architecture

```text
Browser: Next.js / React / CopilotKit
  |-- Supabase Auth and database APIs
  |-- /api/copilotkit --> LangGraph server --> OpenAI models
  |-- /api/cache ------> Redis
  `-- /api/blob/upload -> Vercel Blob
```

- **Frontend:** Next.js 15.3.3 App Router, React 19, TypeScript, Tailwind CSS 4, and Lucide icons.
- **AI integration:** CopilotKit connects frontend actions and shared state to LangGraph agents. Python agents use LangChain and OpenAI models, primarily `gpt-4o-mini`.
- **Persistence:** Supabase stores user profiles, health records, preferences, recipes, and insights. Database SQL includes row-level security policies.
- **Supporting services:** Redis provides caching; Vercel Blob supports uploads and cloud exports.

### Agent graphs

[agent/langgraph.json](agent/langgraph.json) exposes these graph IDs:

| Graph ID | Purpose |
| --- | --- |
| `main_coordinator` | Classifies requests and records a target specialist |
| `cycle_tracker` | Cycle tracking assistance |
| `symptom_mood` | Symptom and mood assistance |
| `fertility_tracker` | Fertility tracking assistance |
| `nutrition_guide` | Nutrition assistance |
| `exercise_coach` | Exercise assistance |
| `health_insights` | Health data analysis assistance |
| `lifestyle_manager` | Sleep and stress assistance |
| `menstrual_tracker` | Separate menstrual tracking implementation |
| `shared_state` | Recipe generation and shared recipe state |
| `sample_agent` | Starter agent example |

Graph availability on the Python server does not guarantee that a frontend page selects that graph. The Next.js runtime has its own registration list in [src/app/api/copilotkit/route.ts](src/app/api/copilotkit/route.ts).

## Local setup

### Prerequisites

- Node.js compatible with Next.js 15 and npm; Node.js 20 is a suitable baseline.
- Python 3.12, matching `agent/langgraph.json`. The Poetry metadata specifies Python `>=3.10,<3.13`.
- A Supabase project and an OpenAI API key with access to the models referenced in `agent/`.
- Redis for the cache integration, and Vercel Blob credentials if using uploads.

Run frontend commands from the repository root and Python commands from `agent/`.

### 1. Install frontend dependencies

```powershell
npm ci
Copy-Item .env.example .env.local
```

On macOS or Linux, use `cp .env.example .env.local` for the copy step.

### 2. Configure the frontend

Edit `.env.local`. The supplied template omits Supabase and Redis configuration, so add those values explicitly:

```dotenv
NEXT_PUBLIC_SUPABASE_URL=https://YOUR_PROJECT.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=YOUR_SUPABASE_ANON_KEY

OPENAI_API_KEY=YOUR_OPENAI_API_KEY
LANGGRAPH_DEPLOYMENT_URL=http://localhost:2024
NEXT_PUBLIC_COPILOTKIT_RUNTIME_URL=http://localhost:3000/api/copilotkit
NEXT_PUBLIC_COPILOTKIT_AGENT_NAME=main_coordinator
NEXT_PUBLIC_COPILOTKIT_AGENT_DESCRIPTION=Women's health assistant
NEXT_PUBLIC_MENSTRUAL_AGENT_NAME=menstrual_tracker
NEXT_PUBLIC_MENSTRUAL_AGENT_DESCRIPTION=AI Menstrual Cycle Tracker

REDIS_URL=redis://localhost:6379
```

Additional configuration:

| Variable | When needed |
| --- | --- |
| `LANGSMITH_API_KEY` | Authentication for a hosted LangGraph deployment; also included in the Python environment template |
| `BLOB_READ_WRITE_TOKEN` | Server-side Vercel Blob uploads |
| `SUPABASE_SERVICE_ROLE_KEY` | If using the privileged helper in `src/lib/supabase/server.ts`; keep server-side only |
| `NEXT_PUBLIC_COPILOT_API_KEY` | Optional Copilot Cloud configuration used by the `/copilotkit` demo layout |
| `NEXT_TELEMETRY_DISABLED=1` | Optional Next.js telemetry setting |

Most health pages use the relative runtime URL `/api/copilotkit` directly. Changing `NEXT_PUBLIC_COPILOTKIT_RUNTIME_URL` alone does not change those pages. Remove unused placeholder credentials from the copied template.

Only public configuration belongs in `NEXT_PUBLIC_*` variables. Keep OpenAI keys, service-role keys, LangSmith keys, and Blob tokens in server-side environment variables. Restart the development server after configuration changes.

### 3. Prepare Supabase

Use a fresh development project and inspect the SQL files before executing them in the Supabase SQL Editor. These are manual setup and repair scripts, rather than an automated migration system.

| File | Role and considerations |
| --- | --- |
| `1-database-setup.sql` | Base tables, policies, and signup setup |
| `2-database-fix.sql` | Replacement signup function and trigger; apply if needed |
| `3-rls-policy-fix.sql` | Disables RLS on profiles and preferences; do not apply as a routine setup step |
| `4-database-schema-extension.sql` | Additional dashboard and insights schema |
| `5-fix-sample-data.sql` | Optional sample data for the most recently created profile; requires a user |
| `6-fertility-tables.sql` | Fertility tables; drops existing fertility tables before recreating them |
| `7-recipe-tables.sql` | Recipes and collections |
| `8-add-avatar-url.sql` | Profile avatar column |
| `9-add-unique-constraints.sql` | Removes duplicate records and adds uniqueness constraints |
| `10-nutrition-focus-table.sql` | Nutrition focus storage |

Start with the base schema, then review and apply the schema extensions in numeric order as appropriate. Repair and sample-data scripts are conditional. Do not bulk-run the directory against an existing database: some scripts delete data, drop tables, or weaken access controls.

Configure Supabase Auth for `http://localhost:3000`, with the relevant `/auth/callback` and `/auth/reset-password` redirect URLs allowed. Sign up through the application and complete email confirmation if enabled. Verify that signup creates a profile and preferences.

### 4. Start the Python agent server

In a separate terminal:

```powershell
cd agent
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
Copy-Item .env.example .env
```

On macOS or Linux, activate with `source .venv/bin/activate` and copy with `cp .env.example .env`.

Set `OPENAI_API_KEY` in `agent/.env`, and set `LANGSMITH_API_KEY` if required by your deployment or tracing configuration. The root `.env.local` is separate from the agent environment.

```powershell
langgraph dev --host 127.0.0.1 --port 2024
```

Run this from `agent/` so the CLI finds `langgraph.json` and its `.env` file. Keep this process running, and point the frontend's `LANGGRAPH_DEPLOYMENT_URL` at its address.

The repository also includes Poetry metadata and a lockfile, but their dependency versions differ from `requirements.txt`. The commands above use the pinned requirements; avoid mixing the two dependency sets in one environment.

`sample_agent/demo.py` is an alternative FastAPI example exposing only `sample_agent`. It is not the full multi-graph backend expected by the current Next.js runtime.

### 5. Start the frontend

With Redis available at the configured URL, open another terminal at the repository root:

```powershell
npm run dev
```

Open [localhost:3000](http://localhost:3000), sign in, and try creating a record. Missing Redis is handled by cache callers as a fallback, but connection attempts can delay requests. Vercel Blob is needed only for features that upload files.

## Development and verification

| Command | Purpose |
| --- | --- |
| `npm run dev` | Start the Next.js development server |
| `npm run build` | Build the frontend for production |
| `npm start` | Serve the completed production build |
| `npm run lint` | Run the configured `next lint` script |
| `npx tsc --noEmit --incremental false` | Check TypeScript separately |
| `npx eslint .` | Run ESLint directly using the repository configuration |

No automated test suite or test script is included. For a manual smoke test, verify signup and password recovery, create and reload a health record, and exercise the AI sidebar with the Python backend running. Verify access isolation with separate user accounts before using real data.

## Current limitations

- **Coordinator handoff is incomplete.** `main_coordinator` sets routing state and returns a routing message; it does not invoke specialist subgraphs.
- **Recipe agent registration needs alignment.** `/recipe` selects `shared_state`, which exists in `langgraph.json` but is absent from the runtime's default agent list. Register it in the runtime before expecting that page's AI integration to work.
- **Export and import are demonstrations.** `useDataExportImport` generates mock records for export and simulates import processing. These controls do not provide a verified backup and restore workflow for stored health records.
- **API authorization is incomplete.** The cache endpoints accept caller-supplied keys and deletion patterns without session checks. Blob upload authentication is commented out, and uploads default to public access. The client-side page gate does not protect these API routes.
- **Database security depends on the scripts applied.** In particular, `3-rls-policy-fix.sql` disables protection for profiles and preferences. Review policies and restore appropriate RLS before exposing user data.
- **Build configuration needs consolidation.** Both `next.config.ts` and `next.config.mjs` exist with different settings. The `.mjs` file suppresses build-time lint and type errors and references `@svgr/webpack`, which is absent from `package.json`. Confirm the configuration in use and run checks separately.
- **Lint command needs verification.** `package.json` uses `next lint` with Next.js 15.3.3. If that command is unavailable in the installed version, use `npx eslint .` directly.
- **Some views contain seeded, sample, or calculated values.** Treat scores, correlations, predictions, and generated guidance as application outputs rather than validated clinical findings. Medical and fertility guidance requires independent validation before clinical use.

## Deployment

Deploy the Next.js application and Python LangGraph backend separately. Configure the frontend with the deployed graph URL and required credentials, provision Supabase and Redis, and configure Blob storage if uploads are enabled. Update Supabase Auth redirect settings for the deployed domain.

[vercel.json](vercel.json) supplies a 60-second API function duration setting. It does not deploy the Python agents or provision the supporting services. Resolve the integration and access-control issues above before making a deployment available to users.

## Repository layout

```text
agent/                 Python graphs, dependencies, and LangGraph configuration
  main_coordinator/    Request classification and routing state
  *_agent/             Specialist and example agents
database/              Manual SQL setup, schema extensions, and repair scripts
public/                Static assets and web manifest
src/
  app/                 Next.js pages, layouts, and API routes
  components/          Feature interfaces and shared UI
  hooks/               State, database operations, auth, and CopilotKit actions
  lib/                 Supabase, Redis, Blob, and utility integrations
  constants/           Navigation, defaults, and feature configuration
  types/               Shared TypeScript types
  utils/               Calculations, validation, and helpers
```

## Contributing and support

See [CONTRIBUTING.md](CONTRIBUTING.md) for contribution guidelines, [SUPPORT.md](SUPPORT.md) for help, and [SECURITY.md](SECURITY.md) for vulnerability reporting. Participation follows the [Code of Conduct](CODE_OF_CONDUCT.md). Project changes are recorded in [CHANGELOG.md](CHANGELOG.md).

Licensed under the [MIT License](LICENSE), copyright 2025 Chan Meng.
