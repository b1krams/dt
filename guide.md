# Architectural Directive: Serverless Full-Stack AI Web Application ($0 Idle & Free-Tier Maximized)

You are an expert principal software architect and full-stack engineer. Build a production-grade, scale-to-zero serverless web application named **[PROJECT_NAME]** adhering strictly to the architectural standards, Always-Free tier guardrails, edge-routed security, streaming LLM engine patterns, and reactive frontend specifications detailed below.

---

## 1. Application Specification (Customizable)

- **Application Name**: [PROJECT_NAME]
- **Core Mission**: [DESCRIBE THE APPLICATION PURPOSE AND VALUE PROPOSITION]
- **Primary User Workflow**:
  - **Inputs**: [DESCRIBE USER INPUTS: e.g., prompt, document/file upload via S3 Presigned URL, multi-parameter configuration form]
  - **Processing Pipeline / Actions**: [DESCRIBE PIPELINE: e.g., real-time streaming analysis, multi-stage reasoning, extraction, agentic orchestration, schema validation]
  - **Outputs & Deliverables**: [DESCRIBE OUTPUTS: e.g., live streaming token/event output, dynamic dashboard artifacts, structured JSON results, exportable reports]

---

## 2. Invariant Architectural Directives (ADRs)

You MUST strictly enforce these 9 architectural directives:

### ADR-01: True Scale-to-Zero & Always-Free Tier Architecture ($0 Cost Target)

- **Compute (1M Free Requests / 400k GB-sec Monthly)**:
  - Execute exclusively on ephemeral **AWS Lambda** running **Python 3.13** on **ARM64 (Graviton2/3)** architecture (20% cheaper and faster than x86_64).
  - Fixed servers (EC2), containers (ECS/EKS/Fargate), VPC NAT Gateways, ElastiCache, and RDS/Aurora are strictly prohibited. Lambdas must run in the default VPC-less AWS network to avoid ENI/NAT costs.
- **Database (25 GB + 25 RCU/WCU Always-Free Optimization)**:
  - Utilize a single **Amazon DynamoDB** table.
  - **Free-Tier Rule**: Configure DynamoDB in `PROVISIONED` billing mode with **10 RCU and 10 WCU** (staying well underneath AWS's 25 RCU / 25 WCU Always-Free tier limit) with auto-scaling disabled, OR expose a toggle in `sst.config.ts` to switch between `PROVISIONED` (100% Free Tier) and `PAY_PER_REQUEST` (On-Demand, $0 idle). Do not create more than 2 Global Secondary Indexes (GSIs).
- **Storage (5 GB S3 Standard Free Tier + Zero Orphan Costs)**:
  - Raw file uploads, large payloads, and generated export artifacts must reside in **Amazon S3**.
  - S3 buckets must enforce automated Lifecycle Rules:
    1. `AbortIncompleteMultipartUpload` after **1 day** (prevents hidden charges from failed uploads).
    2. Expire temporary/scratch artifacts under `tmp/` after **7 days**.
- **CDN & Edge Routing (1 TB Egress + 10M Requests Monthly Free Tier)**:
  - Static web asset delivery and API edge routing must run via **Amazon CloudFront** using Origin Access Control (OAC) and PriceClass `PriceClass_100` (cheapest edge footprint).

### ADR-02: Zero-Cost Secrets, Encryption, & Log Hygiene (Anti-Leak Guardrails)

- **No Paid Secret Managers**:
  - **AWS Secrets Manager** (0.40/secret/month)and**KMSCustomerManagedKeys**(1.00/key/month) are strictly prohibited.
  - Store API keys and sensitive configuration via **AWS Systems Manager (SSM) Parameter Store (Standard Tier — $0.00)** fetched and cached at cold start, or encrypted Lambda environment variables injected at deploy time via `.env` / SST Secrets backed by SSM.
  - Use default AWS-managed encryption keys (`SSE-S3` for S3 and AWS-owned keys for DynamoDB) at $0 cost.
- **CloudWatch Log Cap (5 GB Always-Free Tier)**:
  - Every Lambda function log group in `sst.config.ts` MUST explicitly set `retention: "1 week"` (or `"3 days"`). Indefinite log retention is prohibited.
  - Sample debug logs at 10% in production; emit metrics strictly via **CloudWatch Embedded Metric Format (EMF)** JSON printed to `stdout` rather than paid `PutMetricData` API calls.

### ADR-03: SST v3 (Ion) Infrastructure with Single-Domain Routing

- Infrastructure as Code (IaC) must be provisioned via `sst.config.ts` using **SST v3 (Ion engine powered by Pulumi)**.
- A single **CloudFront distribution** must route static frontend assets from S3 and proxy all `/api/*` endpoints directly to **Lambda Function URLs** (`streaming: true` for LLM endpoints).
- Single-domain routing eliminates CORS preflight (`OPTIONS`) latency/invocations and bypasses API Gateway's 29-second timeout and 1.00-3.50/million request surcharge, unlocking full 15-minute Lambda execution streams at $0 routing cost.

### ADR-04: Edge-Guarded Auth, SigV4 Payload Hashing, & Economic Rate Limiting

- **User Authentication (Cognito Free Tier)**:
  - Manage user identities via an **AWS Cognito User Pool** configured on the **Lite tier** (preserving the 10,000+ MAU Always-Free tier; do not enable paid Advanced Security/WAF features).
- **Edge Perimeter Protection & The OAC POST Body Rule**:
  - CloudFront must connect to Lambda Function URLs using **Origin Access Control (OAC)** with SigV4 request signing (`lambda:InvokeFunctionUrl` and `lambda:InvokeFunction` permissions granted to `cloudfront.amazonaws.com`).
  - Lambda Function URLs must set `AuthType: AWS_IAM`. Direct requests to `.lambda-url.*` hostnames are rejected at the AWS edge at $0 cost.
  - **CRITICAL OAC POST/PUT REQUIREMENT**: When CloudFront OAC signs `POST` or `PUT` requests to Lambda Function URLs, AWS requires the client to send the SHA-256 hash of the payload in the `x-amz-content-sha256` header. The frontend API client (`frontend/src/lib/api.ts`) MUST compute this hash via the browser's native Web Crypto API (`crypto.subtle.digest('SHA-256', body)`) and attach it to every `POST`/`PUT` request.
- **Application-Layer Economic Security (Zero-Cost Rate Limiting)**:
  - AWS WAF ($5/mo base + per-rule fees) is strictly prohibited.
  - Prevent LLM cost exhaustion via an **ASGI Middleware Rate Limiter** running inside the Lambda Web Adapter.
  - Enforce tumbling window quotas (e.g., max 60 requests / 5 mins per User ID or `CloudFront-Viewer-Address` IP) using a single atomic DynamoDB `UpdateItem` (`ADD request_count :inc`, `SET ttl = if_not_exists(ttl, :exp)`) against `PK: RATELIMIT#<id>`, `SK: WINDOW#<5m_bucket>`.
  - DynamoDB **Time-To-Live (TTL)** must automatically prune expired rate-limit keys at zero compute cost.
  - Reject over-quota requests with `HTTP 429 Too Many Requests` and a `Retry-After` header before invoking any upstream LLM.
- **Backend Claims Verification**:
  - Verify `Authorization: Bearer <JWT>` RS256 signatures in-memory using a module-scoped cached JWKS client (`PyJWT` / `aws-lambda-powertools`) so cold/warm invocations make zero paid network calls to Cognito.
  - Enforce tenant/domain restrictions via an `ALLOWED_EMAIL_DOMAINS` environment variable.

### ADR-05: High-Performance Streaming LLM Engine & Free-Tier Model Cascade

- **Python Response Streaming Engine**:
  - Configure streaming Lambdas with `InvokeMode: RESPONSE_STREAM` and attach the official **AWS Lambda Web Adapter (LWA)** ARM64 layer (`arn:aws:lambda:${region}:753240598075:layer:LambdaAdapterLayerArm64:24`) with `AWS_LWA_INVOKE_MODE: "response_stream"` and `AWS_LWA_READINESS_CHECK_PATH: "/api/health"`.
  - Run a lightweight **Starlette** ASGI app returning `StreamingResponse(media_type="text/event-stream")` with headers `Cache-Control: no-cache, no-transform` and `X-Accel-Buffering: no`.
- **Non-Blocking Network Layer (Zero SDK Bloat)**:
  - Use a module-scoped singleton `httpx.AsyncClient(http2=True)` for connection pooling across warm Lambda invocations.
  - Heavy vendor SDKs (`openai`, `anthropic`, `google-genai`, `langchain`, `litellm`) are **prohibited** to keep deployment packages under 15 MB and cold starts under 400ms. Write lean, direct HTTP/SSE adapters in `core/llm/adapters.py`.
- **Free-Tier LLM Cascade & Failover Resilience**:
  - Implement an automated provider cascade in `core/llm/engine.py` that prioritizes zero-cost / high-speed free-tier APIs before falling back to paid endpoints on `HTTP 429` or `5xx`:
    1. **Primary (Free/Ultra-Fast Tiers)**: Google Gemini API Free Tier (`gemini-2.5-flash` / `gemini-2.5-pro`), Groq Cloud Free Tier, or OpenRouter `:free` models.
    2. **Secondary (Paid Fallback)**: OpenAI / Anthropic / Bedrock via direct REST.
- **Lambda-Safe Telemetry Flushing (No Frozen BackgroundTasks)**:
  - Because AWS Lambda freezes the execution environment the exact millisecond a `StreamingResponse` generator closes, **do NOT use Starlette `BackgroundTasks` for critical writes**.
  - Instead, flush token usage, latency, and state updates to DynamoDB and emit CloudWatch EMF JSON logs inside the `finally:` block of the async SSE generator *immediately before* yielding the closing `event: done` SSE frame.

### ADR-06: Native Structured Outputs & Strict Schema Contracts

- Every machine-consumed structured output or tool call must conform to a canonical JSON Schema in `schemas/*.schema.json`.
- Backend adapters must enforce native provider structured output contracts:
  - OpenAI / Groq / OpenRouter: `response_format: { "type": "json_schema", "json_schema": { "strict": true, ... } }`
  - Gemini REST API: `generationConfig: { "responseMimeType": "application/json", "responseSchema": ... }`
  - Anthropic REST API: Tool-use schema constraint with forced `tool_choice`.
- Validate all incoming payloads and outgoing structured completions against compiled **Pydantic v2** models (`core/llm/models.py`).

### ADR-07: High-Fidelity Reactive SPA (Vite + Svelte 5 Runes + Tailwind CSS v4)

- **Stack & Zero-SSR Static Export**:
  - Scaffold the frontend as a pure static Single Page Application (SPA) using **Vite**, **Svelte 5** (strictly Runes syntax: `state`,`state.raw`, `derived`,`effect`, `props`—legacySvelte4`exportlet`,`:`, and `writable()` stores are prohibited), and **Tailwind CSS v4** (`@tailwindcss/vite`).
  - Build to pure static assets (`dist/`) served from S3 via CloudFront:
    - Hashed assets (`/assets/*`): `Cache-Control: public, max-age=31536000, immutable`.
    - Entry point (`/index.html`): `Cache-Control: public, max-age=0, must-revalidate` (enabling instant zero-cost deploys without paid CloudFront invalidations).
  - Configure CloudFront SPA fallback routing (`403`/`404` -> `/index.html`) strictly scoped to non-`/api/*` paths so backend JSON errors are never masked by HTML.
- **Zero-SDK Cognito PKCE Auth & Pre-Stream Token Refresh**:
  - Heavy frontend auth SDKs (`aws-amplify`, `amazon-cognito-identity-js`) are prohibited.
  - Implement a lightweight OAuth 2.0 PKCE client using the native Web Crypto API (`crypto.subtle`) against Cognito endpoints, storing tokens securely and automatically refreshing the JWT if it is within **5 minutes of expiration** before initiating any SSE stream or S3 presigned upload.
- **Stream Consumption, OAC Hash Signing & Idempotency**:
  - Consume SSE streams via `fetch()` using `response.body.pipeThrough(new TextDecoderStream()).getReader()` to support `POST` bodies and custom headers.
  - Every `POST`/`PUT` request in `frontend/src/lib/api.ts` must attach:
    1. `Authorization: Bearer <JWT>`
    2. `x-amz-content-sha256`: Hex-encoded SHA-256 digest of the serialized request body (`crypto.subtle.digest('SHA-256', body)`), satisfying CloudFront OAC SigV4 requirements.
    3. `x-idempotency-key`: A `crypto.randomUUID()` generated per logical user action (reused across transient retries to respect ADR-09 DynamoDB locks).
  - Bind an `AbortController` to each active stream to support instant user cancellation and clean up reader locks inside `finally` blocks.
- **60fps Render Protection & Incremental Healing**:
  - Route incoming SSE frames (`data:`, `event:`) through a deterministic line-buffer in `frontend/src/lib/stream_parser.ts` so partial multi-byte UTF-8 characters, split SSE frames, and unclosed Markdown code fences or partial JSON tokens never corrupt the UI.
  - **Frame-Rate Batching**: Accumulate high-velocity token deltas (e.g., Groq/Gemini Flash at 200+ tok/s) in a non-reactive buffer and flush updates into Svelte `$state` runes via `requestAnimationFrame` (max 60 updates/sec) to prevent DOM layout thrashing.
  - Use `$state.raw()` in `stores.svelte.ts` for large read-only structured outputs and artifact payloads to bypass deep JavaScript Proxy overhead.
- **Direct S3 Presigned I/O & Economic UX Guards**:
  - Upload files directly to S3 Presigned URLs using `XMLHttpRequest` (`xhr.upload.onprogress`) for reactive progress tracking after client-side MIME/size validation.
  - When consuming offloaded artifacts (> 350 KB per ADR-09), transparently fetch the presigned S3 GET URL and decompress gzip streams client-side via the native browser `DecompressionStream('gzip')` API.
  - On `HTTP 429 Too Many Requests`, parse the `Retry-After` header and render a reactive cooldown timer in `WorkflowConsole.svelte` that locks submission until the rate-limit window resets.


### ADR-08: First-Class Local DX & Automated Type Synchronization

- Local frontend development must run via `vite dev` with HMR, proxying `/api` calls.
- Backend development must leverage **SST v3 Live Lambda (`sst dev`)**, routing cloud invocations to local Python processes.
- Manage Python environments and lockfiles strictly with **`uv`** (`pyproject.toml` + `uv.lock`).
- Provide `scripts/generate_types.sh` to compile `schemas/*.schema.json` into Python Pydantic v2 models (`datamodel-code-generator`) and TypeScript interfaces (`json-schema-to-typescript`).

### ADR-09: Single-Table State Management & S3 Presigned Offloading

- **Idempotency Guarantees**:
  - Mutating requests must pass an `x-idempotency-key` header.
  - Lock executions via atomic conditional writes in DynamoDB (`ConditionExpression="attribute_not_exists(PK)"`) with a 24-hour TTL.
- **DynamoDB Single-Table Design**:
  - Store all application entities in a single DynamoDB table using composite keys (`PK`, `SK`) and a single overloaded GSI (`GSI1PK`, `GSI1SK`):
    - User Profile: `PK: USER#<user_id>`, `SK: PROFILE`
    - Workflow Session: `PK: USER#<user_id>`, `SK: WORKFLOW#<ulid>`
    - Execution Events: `PK: WORKFLOW#<ulid>`, `SK: EVENT#<timestamp>`
    - Rate Limits: `PK: RATELIMIT#<id>`, `SK: WINDOW#<epoch_bucket>` (with `ttl`)
    - Idempotency Locks: `PK: IDEMPOTENCY#<key>`, `SK: LOCK` (with `ttl`)
- **Large Payload & File Upload Offloading (> 350 KB Safeguard)**:
  - Never route file uploads through Lambda (avoiding Lambda's 6 MB payload ceiling and execution billing). Generate **S3 Presigned PUT/POST URLs** with strict size conditions (`content-length-range`) so the browser uploads directly to S3.
  - Any state record or LLM output exceeding 350 KB must be compressed (`gzip`) and offloaded to S3, storing only the S3 URI, SHA-256 digest, and summary metadata in DynamoDB (preventing `ItemSizeLimitExceeded` 400 KB errors).

---

## 3. Canonical Repository Layout

Scaffold the project matching this exact layout:

```text
[PROJECT_NAME]/
├── package.json                 # SST v3, Vite, Svelte 5, Tailwind CSS v4 dependencies
├── pyproject.toml               # Python 3.13 deps managed via uv (starlette, uvicorn, aws-lambda-powertools, httpx[http2], pydantic, boto3, pyjwt[crypto], ruff)
├── sst.config.ts                # SST v3 (Ion) IaC (CloudFront OAC, DynamoDB Free-Tier, Cognito Lite, S3 Lifecycle, LWA Lambdas, 7-day Log Retention)
├── .env.example                 # Template for LLM keys (Gemini, Groq, OpenAI, Anthropic) & tenant settings
├── .gitignore
├── .vscode/
│   ├── launch.json              # 1-Click debug configs (Vite dev, SST dev, pytest)
│   └── settings.json            # Workspace paths, Python analysis, Ruff formatter settings
├── AGENTS.md                    # Operational rules and architectural constraints for AI coding agents
├── README.md                    # Architecture guide, Free-Tier checklist, quickstart, and deploy manual
├── scripts/
│   └── generate_types.sh        # Executes datamodel-codegen (Pydantic v2) & json-schema-to-typescript
├── schemas/                     # Single source of truth JSON Schema contracts
│   ├── request.schema.json      # Incoming API request payload contract
│   └── response.schema.json     # Canonical structured output JSON schema contract
├── core/                        # Shared Python library
│   ├── __init__.py
│   ├── config.py                # Env/SSM config loader with cold-start caching
│   ├── auth.py                  # Cached Cognito RS256 JWKS verification & domain enforcement
│   ├── db.py                    # DynamoDB Single-Table repository, atomic rate-limiter, & S3 presigner/offloader
│   ├── telemetry.py             # Zero-cost CloudWatch EMF structured metric logger
│   └── llm/
│       ├── engine.py            # Multi-provider streaming orchestrator with free-tier cascade & failover
│       ├── models.py            # Auto-generated Pydantic v2 models (DO NOT EDIT MANUALLY)
│       └── adapters.py          # Zero-SDK httpx HTTP/2 SSE streaming adapters (Gemini, Groq/OpenAI, Anthropic)
├── lambdas/
│   ├── stream_worker/
│   │   ├── main.py              # Starlette ASGI app (LWA StreamingResponse, RateLimit Middleware, inline finally-block telemetry)
│   │   └── run.sh               # Executable (chmod +x) LWA bootstrap script launching uvicorn
│   └── api_handler/
│       └── handler.py           # Synchronous CRUD & S3 Presigned URL Lambda handler
├── frontend/                    # Vite + Svelte 5 Runes + Tailwind CSS v4 SPA
│   ├── index.html
│   ├── vite.config.ts           # Vite config with @tailwindcss/vite & local /api proxy
│   ├── svelte.config.js
│   └── src/
│       ├── app.css              # Tailwind CSS v4 entry (@import "tailwindcss";)
│       ├── main.ts              # SPA mount bootstrap
│       ├── App.svelte           # Root application orchestrator
│       ├── lib/
│       │   ├── crypto.ts        # Web Crypto SHA-256 helper for CloudFront OAC x-amz-content-sha256 header
│       │   ├── api.ts           # Fetch ReadableStream client with OAC hashing, JWT injection, & AbortController
│       │   ├── stream_parser.ts # Line-buffered SSE parser, markdown fence healer, & incremental JSON repair
│       │   ├── stores.svelte.ts # Svelte 5 Runes reactive state (state,derived)
│       │   └── types.ts         # Auto-generated TypeScript interfaces (DO NOT EDIT MANUALLY)
│       └── components/
│           ├── Header.svelte
│           ├── WorkflowConsole.svelte
│           ├── ArtifactViewer.svelte
│           └── MetricsBar.svelte
└── tests/
    ├── test_auth.py
    ├── test_rate_limiter.py
    ├── test_llm_engine.py
    └── test_schema_conformance.py


