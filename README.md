# Crypto Fabric — Modular GCP Architecture & Control Center
> Crypto Fabric is a profitability-first automation platform for digital-asset operators. The system pairs a guided **Control Center** with service templates that keep staking, trading, and infrastructure workloads compliant with the same guardrails. Teams start in a zero-cost developer mode, test the full experience locally, and then promote to Cloud Run when they are ready to generate yield.

## Business Plan Overview
[Busines Plan Overview](./BUSINESS_PLAN.md) (BUSINESS_PLAN.md)

## Business overview

- **Single orchestration plane** – Centralizes onboarding, guardrails, and rollout policies across dozens of profit-seeking services without bespoke scripting.
- **Native mobile superpowers** – Ships white-labeled iOS and Android binaries for both Super Admins and client tenants, generated straight from Firebase Remote Config + Expo profiles. Super Admins can promote new configurations and trigger branded builds directly from the Firebase console or companion mobile app, keeping releases in lockstep with profitability guardrails.
- **Profitability telemetry out of the box** – Every module reports revenue, spend, and profit indices back to the Control Center, so new strategies compete on actual margins instead of projections.
- **Promotion-ready workflow** – Operators launch the Expo-powered Super Admin (`yarn workspace crypto-fabric-admin start --web`), evaluate services against live Firebase data, push native build toggles from their phones, and toggle “Promote to production” once the stack passes profitability and guardrail checks.
- **Real-time profit telemetry** – Mobile dashboards surface the same profitability, burn, and guardrail scores as the Control Center so field teams can pivot strategies with current margins instead of lagging reports.

### What makes Crypto Fabric different

1. **Designed for regulated teams** – Secrets stay in Google Secret Manager, IAM/IAP wrap the hosted dashboard, and manifests are policy-checked before rollout.
2. **Profit-aware automation** – The orchestrator only scales when profitability indices stay positive, reducing costly experiments.
3. **Two-speed delivery** – Development stays Python-only and bill-free, while production uses Cloud Run + Artifact Registry with the same manifests.

## Operating modes

- **No-Cost Dev** – The launcher defaults to `DEV_NO_COST=true` and swaps Google Cloud APIs for local adapters (Secret Manager stubs, Pub/Sub emulator, mock AI providers). Developers can run the entire wizard without installing gcloud. See [DEVELOPMENT.md](DEVELOPMENT.md) for the full playbook.
- **Production** – Flipping the Control Center toggle sets `DEV_NO_COST=false` and `CLOUD_DEPLOY=true`, deploying the curated stack to Cloud Run behind IAP. Policy gates ensure only opted-in environments spend money, and native builds automatically inherit the tenant’s Remote Config payload so mobile experiences stay synchronized with backend rollouts.

## Quick start — launch the Control Center

```bash
yarn install
export EXPO_PUBLIC_FIREBASE_PROJECT_ID="my-firebase-project"
export EXPO_PUBLIC_FIREBASE_API_KEY="<api-key>"
# ... set remaining EXPO_PUBLIC_FIREBASE_* variables or run the Firebase bootstrap script ...
yarn workspace crypto-fabric-admin start --web
```

> **Zero-cost defaults:** The Super Admin workspace honours `DEV_NO_COST=true`, exercising Firebase + Google Cloud APIs without provisioning Cloud Run resources until profitability gates clear. Install Google Cloud SDKs only when you intentionally promote to production.

The Expo development server authenticates with Google IAM and opens the web Control Center. From there you can:

1. **Bootstrap the core footprint** – Provision Artifact Registry, Secret Manager scaffolding, and the telemetry bus needed by every module.
2. **Run service wizards** – Each service card links to its `wizard.py`, collecting credentials, verifying guardrails, and publishing Secret Manager entries before a deploy can proceed.
3. **Launch workloads** – After a wizard succeeds, click **Launch** to apply the Cloud Run manifests and begin streaming profitability metrics to the dashboards.

### Bootstrap the Super Admin Firebase API key

Every tenant uses a dedicated Firebase Web API key that is stored alongside the client record in Firestore. A background service
worker (Firebase Function) issues keys for any client document that does not already have one, and the Expo build pipeline writes
the resolved key into `admin/.env.local` so the React Native web bundle initializes Firebase without manual `.env` editing.

1. Authenticate with a service account that can write to the staging Firestore project (set `GOOGLE_APPLICATION_CREDENTIALS`).
2. Run `node ./scripts/create-default-client.mjs` to create or refresh the default client record and wait for the service worker
   to inject its API key.
3. Run `yarn build` or `yarn start` — both commands copy the Expo workspace and generate `admin/.env.local` using the Firestore
   value, so `EXPO_PUBLIC_FIREBASE_API_KEY` and friends are automatically present during the Expo export.

The script accepts optional flags such as `--client=<id>`, `--name=<label>`, and `--owner-email=<address>` if you want to seed a
different tenant or override metadata. All other Firebase configuration fields fall back to `EXPO_PUBLIC_FIREBASE_*` environment
variables when not provided in Firestore.

### Static hosting artifacts

The Firebase Hosting surface serves three entry points: the public client portal (`/app`), the unlinked Super Admin interface (`/admin`), and a root document that forwards to the client portal. Run `yarn build` before deploying to generate Expo web bundles for both apps and stage the supporting static files under `dist/`:

- `dist/index.html` — branded landing page that redirects the bare domain to `/app/`.
- `dist/sitemap.xml` — exposes `/app/` and `/admin/` to crawlers so the Super Admin URL remains discoverable without being linked from the portal UI.
- `dist/robots.txt` — points search engines at the sitemap.

Set `CF_HOSTING_BASE_URL=https://your-domain.example` during the build to embed the correct absolute URLs in the sitemap and robots file. When unset the tooling falls back to `https://localhost`, which keeps emulator runs hermetic.

## Architecture snapshot

- **ProfitService & registry (`core/`)** – Provides the lifecycle contract for every workload and auto-discovers manifests, environment defaults, and `wizard.py` routers so the Control Center can render them dynamically.
- **Guardrails & telemetry (`core/costs.py`, `core/metrics.py`)** – Model profitability, enforce scaling budgets, and surface dashboards without bespoke wiring.
- **Treasury automations (`core/treasury.py`)** – Handle revenue sweeps, ETH payouts, and reinvestment policies once strategies are profitable.
- **GCP integration (`core/gcp.py`)** – Supplies Cloud Run and Artifact Registry clients, falling back to local adapters whenever `DEV_NO_COST=true`.

## Core service stack

The curated control-plane lives in `services/core/core.yml`:

| Service | Description |
| --- | --- |
| `orchestrator` | Plans and schedules profitability-aware workloads on a Tier-A GKE Autopilot cluster with guardrails sourced from `core/costs.py`. |
| `command-center` | IAM-aware Cloud Run portal that exposes the registry state, guardrail status, and documentation links via the FastAPI app in `services/core/command-center/command-center.py`. |
| `cost-exporter` | Normalises Cloud Billing data into unit-cost metrics via `services/core/cost-exporter/cost-exporter.py`. |
| `telemetry` | Bridges exporter data into Cloud Monitoring dashboards and alerting policies via `TelemetryAggregator` + `MetricsPublisher`. |

Each service directory contains a manifest, tracked `.env`, example secrets contract, implementation module, and `wizard.py` onboarding flow so new workloads stay compliant with the same governance.

## Mobile-first value creation

- **White-labeled client apps** – Every tenant receives a branded build assembled from Firestore + Remote Config metadata. The Control Center packages assets, color palettes, and on-chain hooks so operators monetize faster without touching native code.
- **Super Admin command suite** – The Super Admin mobile app mirrors the web Control Center, letting founders approve guardrail overrides, issue promotions, and stream real-time profit telemetry while on the move.
- **Firebase-native distribution** – Using the Firebase console and companion admin app, teams schedule over-the-air config pushes, queue App Store / Play Store submissions, and roll back missteps instantly. This workflow eliminates expensive mobile DevOps cycles and is a major differentiator for investors evaluating scale efficiency.
- **Investor-grade analytics** – Dashboards blend profitability indices, capital efficiency, and customer engagement so financial projections mirror live performance. Mobile-first alerts tie revenue spikes directly to service catalogs, keeping forecasts grounded in current unit economics.
- **YachtOffice + YOToken flywheel** – Crypto Fabric powers the upcoming **YachtOffice** marketplace, where profits minted by tenant strategies mint **YOToken**, the first crypto asset with intrinsic value derived from tokenized cash flows flowing through this platform. Early investors capture upside from every automated deployment.

### Profit flywheel & YachtOffice alignment

- **Realtime profitability receipts** – Profit indices, guardrail audits, and treasury movements land in Firestore and Cloud Monitoring streams, giving investors day-to-day telemetry without custom dashboards.
- **Treasury-backed tokenization** – The upcoming **YachtOffice** initiative extends these cash flows to **YOToken**, positioning it as the first crypto instrument with verifiable intrinsic value because it is literally backed by tokenized profits from Crypto Fabric-managed tenants.
- **Compounding services catalog** – As new profit engines join the marketplace, Super Admins can bundle them into white-labeled mobile experiences, expanding the revenue share that ultimately flows into YOToken while keeping operating costs flat thanks to automation.

## Modular service catalog

Legacy workloads from the monolithic orchestrator have been ported into dedicated service directories so they can be composed
into new stacks without reintroducing tight coupling. The registry discovers the following modules under `services/`:

| Directory | Manifest name | Default tier | Primary focus |
| --- | --- | --- | --- |
| `services/opportunistic/aave-liquidator/` | `aave-liquidator` | `tier-c` | Opportunistic liquidations with on-chain guardrails. |
| `services/akash/akash/` | `akash-provider` | `tier-b` | Leasing compute capacity on Akash with region-aware scaling. |
| `services/eigen/eigen/` | `eigen-operator` | `tier-a` | EigenLayer restaking operator orchestration. |
| `services/forta/forta-node/` | `forta-node` | `tier-b` | Forta threat detection node with allowlist management. |
| `services/trading/freqtrade/` | `freqtrade` | `tier-b` | Freqtrade trading cluster with exchange rotation. |
| `services/hopr/hopr/` | `hopr-node` | `tier-b` | HOPR privacy node with bandwidth controls. |
| `services/trading/hummingbot/` | `hummingbot` | `tier-b` | Market making automation sourcing configs from Cloud Storage. |
| `services/opportunistic/stat-arb-l2/` | `stat-arb-l2` | `tier-b` | Cross-chain arbitrage router tuned for gas ceilings. |
| `services/lava/provider-base/` | `provider-base` | `tier-b` | Lava RPC provider footprint focused on Base. |
| `services/opportunistic/mev-share/` | `mev-share` | `tier-b` | Builder-compatible MEV-Share strategies with bundle caps. |
| `services/nym/nym/` | `nym-gateway` | `tier-b` | NYM gateway bandwidth scheduling and mixnet layering. |
| `services/pocket/pocket/` | `pocket-node` | `tier-b` | Pocket Network validator relays and payout automation. |
| `services/saturn/saturn-node/` | `saturn-node` | `tier-b` | Saturn CDN cache nodes with regional peering. |
| `services/ssv/operator/` | `operator` | `tier-b` | SSV distributed validators with DKG support. |
| `services/storj/storj/` | `storj` | `tier-b` | Storj storage nodes with ingress/egress accounting. |
| `services/core/treasury/reinvestor/` | `reinvestor` | `tier-a` | Capital allocation engine for profitable reinvestment. |

## Cost outlook and pricing context

- **Cloud Run compute** – In `us-central1`, active CPU is billed at **$0.000024 per vCPU-second** and memory at **$0.000002 per GiB-second** after the free tier (240,000 vCPU-seconds and 450,000 GiB-seconds each month). Idle min-instances are cheaper at **$0.0000025 per vCPU-second** but stay disabled in dev mode for zero-cost idle. These figures come directly from the [Google Cloud Run pricing table](https://cloud.google.com/run/pricing).
- **Scenario** – Two 1 vCPU / 2 GiB services running eight hours per day for a month consume 1,728,000 vCPU-seconds and 3,456,000 GiB-seconds. After subtracting the free tier, the monthly spend is roughly **$41.72** (≈$35.71 CPU + $6.01 memory) before network egress. 【911851†L1-L2】
- **Messaging** – Cloud Run includes 2 million requests per month at no additional cost, covering the Control Center’s internal traffic in most deployments.

Because the orchestrator only scales services when their profitability index stays above zero, production rollouts can be evaluated against these concrete budgets inside the Control Center before any toggle is flipped.

## Google Cloud alignment

- **Deployment** – All compute lands on GKE Autopilot or Cloud Run via the helpers in `core/gcp.py`. No local Docker, no
  alternative code paths.
- **Secrets** – Every secret value is referenced via `projects/<project>/secrets/<name>/versions/latest` placeholders. Rotations
  happen in Secret Manager, not in the repo.
- **Telemetry** – Services emit profitability metrics through the shared `ProfitTelemetry` structure. The exporter publishes to
  `metrics.raw.v1`, and the metrics bridge binds those signals into Cloud Monitoring dashboards.
- **Cost discipline** – Manifests expose per-service revenue and spend assumptions so the orchestrator can enforce
  `revenue_per_hour >= spend_per_hour` in rolling windows.

## Contributing new services

1. Create `services/<stack>/<service-name>/`.
2. Add `<service-name>.yml` (or `.yaml`) with a `service.entrypoint` pointing at a module that exposes a `ProfitService`
   subclass. For hyphenated directories, reference the file path (e.g. `services/core/command-center/command-center.py:ClassName`).
3. Add `<service-name>.env` with non-secret knobs and `<service-name>.secrets.example` documenting required Secret Manager URIs.
4. Implement `<service-name>.py` extending `ProfitService` and using `GCPEnvironment` helpers for all provisioning.
5. Implement `wizard.py` subclassing `core.ServiceWizard`. Override `description()` or `apply()` as needed so the Control Center
   can render onboarding forms and persist updates through Secret Manager.
6. Update the stack manifest (e.g. `services/<stack>/<stack>.yml`) to list the new service.

From the Super Admin workspace, open the stack catalog to confirm discovery works, then run the profitability planning flow to validate the deployment plan before promoting services.

For detailed credential workflows, alert routing, and day-two operations, see [RUNBOOK.md](RUNBOOK.md) and [NEXT_STEPS.md](NEXT_STEPS.md).
