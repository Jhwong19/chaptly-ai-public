# ![Serverless AI Data Pipeline](img/cover_page.jpg)
# Chaptly (YouTube Tutor Chatbot)

Chaptly is a conversational learning assistant that turns long-form YouTube lectures into an interactive study companion. Learners paste a video URL, the system fetches transcripts, builds Gemini-powered embeddings, and exposes search/QA/bookmark workflows via a FastAPI backend running on AWS Lambda.

---

## Problem Statement
Educational videos are dense, hard to skim, and often lack structured navigation. Students waste time hunting for concepts, tutors cannot quickly surface key clips, and mobile-friendly quiz experiences are rare. We need a way to index trusted educational channels, ground answers in the source transcript, and expose low-latency APIs that downstream frontends (Hugging Face Spaces, Streamlit, etc.) can call securely.


## Solution Overview
- Fetch YouTube transcripts and chunk them by time, paragraph, or sliding windows.
- Generate sentence embeddings with Gemini and cache the artifacts in S3.
- Answer questions, build bookmarks, and draft quizzes by retrieving top chunks and prompting Gemini chat models.
- Serve everything through a FastAPI app deployed to AWS Lambda + API Gateway.


## High-Level Architecture
![User Flow & System Architecture](img/user_flow_system_architecture.jpg)
```
User UI (Hugging Face Space / Streamlit)
            |
            v
API Gateway (HTTP API) ──> AWS Lambda (FastAPI via Mangum)
            |                         |
            |                         ├── S3 (artifacts, Lambda ZIP)
            |                         ├── DynamoDB (ChaptlyJobs)
            |                         ├── Secrets Manager (HMAC + runtime config)
            |                         └── Gemini APIs (embeddings + LLM answers)
            v
CloudWatch Logs / S3 quota logs
```

## Client–Server Interaction Flow
1. Client sends requests to API Gateway.
2. API Gateway proxies routes to the Lambda function.
3. Mangum boots FastAPI and authenticates the request.
4. For ingestion, the backend pulls transcripts, chunks, embeds via Gemini, and saves artifacts (`chunks.json`, `embeddings.npy`, metadata) to S3.
5. Searches and QA requests reuse cached artifacts; only retrieval + Gemini text generation run at request time.
6. Responses bubble back through API Gateway to the client; logs land in CloudWatch and quota events in S3.


## Infrastructure as Code (Terraform)
- `infra/terraform` defines every managed component (S3 artifact bucket, DynamoDB job table, IAM roles/policies, Lambda, API Gateway, CloudWatch logs) so prod/staging stay in lockstep.
- Applying Terraform (`terraform plan && terraform apply`) reads the latest Lambda ZIP from S3, wires env vars, and applies throttling + permissions without console clicks.
- `terraform state list` should enumerate the same resources (S3 bucket + lifecycle/public-access/versioning helpers, DynamoDB table, IAM role/policies, Lambda + permission, API Gateway pieces, CloudWatch log group) so you can verify no drift or missing imports. Use `terraform state show aws_s3_bucket.artifact_store` (or any other resource) whenever you need to confirm tags, encryption, or lifecycle rules before editing.
- The S3 bucket is versioned, blocked from public access, encrypted (SSE-S3), and configured with lifecycle rules (current objects expire after 30 days, non-current versions after 7) so cached artifacts clean up automatically.
- Inputs such as `lambda_reserved_concurrency`, `artifact_expiration_days` can be overridden via `terraform.tfvars`; outputs surface ARNs/URLs for CI wiring.

Need a refresher? See `infra/terraform/README.md` for a resource-by-resource breakdown, including how to rotate secrets or destroy the stack.


## System Design Goals & Constraints
- **Grounded answers**: every response references timestamped transcript snippets.
- **Minimal infra footprint**: single Lambda/API Gateway + DynamoDB + S3; optional TF toggle for Lambda.
- **Cold-start aware**: package all dependencies (including NLTK `punkt`) in the Lambda ZIP to prevent runtime downloads.
- **Cost aware**: PAY_PER_REQUEST DynamoDB, lifecycle-managed S3 artifacts, and a 512 MB Lambda footprint.


## Approved Channels & Copyright Compliance
- Only pre-approved educational channels may be ingested to avoid redistributing copyrighted material without explicit licenses.
- Approval covers MIT OpenCourseWare, Stanford Online, Harvard, Yale Courses, Khan Academy, Coursera, edX, Udacity, CrashCourse, TED/TED-Ed/TEDx, 3Blue1Brown, Numberphile, Computerphile, SciShow, Veritasium, MinutePhysics, Programming with Mosh, and freeCodeCamp.
- Each ingestion resolves the channel via metadata and rejects anything outside the whitelist with HTTP 403 plus the approved list for user context.
- Legal terms mirror `terms-of-use.md`; clients must confirm rights before submitting unlisted channels.


## Data Model
- **S3**: `s3://chaptly-rag/<video_id>/chunks.json`, `embeddings.npy`, `artifacts.json`, plus cached quizzes/summaries. Lifecycle policies expire unused versions after 30/7 days.
- **DynamoDB (`ChaptlyJobs`)**: partition key `job_id`, TTL on `expires_at`, used to track ingestion jobs/status.


## Backend Execution Model (AWS Lambda + FastAPI)
- FastAPI app (`src/api/main.py`) runs inside Lambda via Mangum; same code can run locally with `uvicorn`.
- `scripts/package_lambda.sh` builds a manylinux/ARM64 ZIP inside the SAM Python 3.11 container, bundles dependencies + `nltk_data`, and writes `dist/chaptly-api.zip`.
- The ZIP is uploaded to S3; Terraform/Lambda pulls from that bucket on deploy.
- Environment variables (set by Terraform) include `ENV`, `S3_BUCKET`, `LOG_LEVEL`, and overrides.

## Performance & Scaling
- Lambda memory: 512 MB, timeout: 29 s, architecture: ARM64.
- Reserved concurrency can be set via `lambda_reserved_concurrency`; default (0) lets AWS manage.
- S3 artifact caching keeps online queries O(k) where k is number of candidate chunks; ingestion is the expensive path and runs asynchronously.
- DynamoDB PAY_PER_REQUEST easily scales with ingestion bursts.

## Latency Profile
- **Ingestion**: 5–45 s dominated by transcript fetch + Gemini embedding batches, but runs as a background task.
- **Search/QA**: typically < 1 s once artifacts exist, because retrieval is local to S3-cached numpy arrays and only one Gemini call (for QA) runs.
- **Cold start**: mitigated by bundling `nltk_data` and removing numpy hard dependencies; expect ~2–3 s on cold invocations.

## Concurrency & Rate Limiting Strategy
- API Gateway stage throttling: burst 50 / steady 25 requests per second.
- Application-level middleware enforces `MAX_REQUESTS_PER_MINUTE` (default 60) with a token-bucket per API key; quota events log to `s3://chaptly-rag-usage/quota-logs/`.
- Future HMAC signature verification (using `chaptly-api-hmac-key`) will allow tamper-proof client-side signing.


## CORS & Hugging Face Origin Control
- This API is intended to power a single Hugging Face Space front-end. Set allowed origins as needed.
- FastAPI mounts a CORS middleware that only accepts origins present in the configuration; matching origin is also enforced by API Gateway stage settings.
- Local development can temporarily set open origins, but production config should always limit to the HF Space (or corporate domains) to prevent key leakage.


## API Authentication
- Clients must include authentication headers on every request; unauthenticated calls receive HTTP 401.


## Idempotency Features
- Ingestion writes artifacts to deterministic S3 keys (`<video_id>/<strategy>.json`) so retries simply overwrite.
- DynamoDB jobs include `job_id` + TTL; replays with the same `job_id` update the same row.
- `/videos/process` can be invoked multiple times safely; the latest chunk set replaces prior ones.

## Metrics & Observability
- CloudWatch log group `/aws/lambda/chaptly-api` retains 14 days (configurable).
- Structured logging includes `request_id`, `video_id`, and error metadata.
- Quota middleware writes JSON events to `s3://chaptly-rag-usage/quota-logs/` for offline analysis.
- Terraform outputs the API Gateway URL, Lambda ARN, and secret names for hooking into dashboards.


## Technology Stack
| Layer | Tools |
|-------|-------|
| Frontend clients | Hugging Face Space, Streamlit, or any app that can call HTTPS |
| Backend | FastAPI, Mangum, Pydantic, AnyIO |
| Transcripts | `youtube_transcript_api`, optional Whisper + FFmpeg for fallbacks |
| Embeddings + LLM | Gemini + Gemini Flash |
| Storage | AWS S3 (artifacts + Lambda ZIP), DynamoDB (jobs) |
| Deployment | Terraform, AWS Lambda, API Gateway, CloudWatch Logs |


## Key Features
- Paste a YouTube URL and ingest transcripts asynchronously.
- Generate bookmarks with section titles, timestamps, and descriptions.
- Ask free-form questions with grounded answers + source timestamps.
- Produce quick quizzes and summaries via Gemini prompts.
- Restrict usage to pre-approved channels; warn on unknown sources.


## Deployment & Packaging (S3 ZIP Workflow)
1. Run `./scripts/package_lambda.sh` to build `dist/chaptly-api.zip` inside the AWS SAM Python 3.11 (ARM64) container. The script pins wheels, prunes tests, and bundles `nltk_data`.
2. Upload the ZIP to S3 (or any bucket/key set via configuration).
3. Apply Terraform (`infra/terraform`) to provision/refresh S3, DynamoDB, IAM, Lambda, and API Gateway. Terraform reads the uploaded ZIP path and updates Lambda accordingly.

### Monitoring YouTube Transcript Fetches
- Tail the Lambda logs (`/aws/lambda/chaptly-api`) and filter for `get_transcript_with_api.blocked` warnings which indicate `youtube_transcript_api` hit `IpBlocked`/`RequestBlocked` errors.
- Quick CLI tail: `aws logs tail /aws/lambda/chaptly-api --since 5m --follow` shows the most recent warnings/errors in real time.
- If you see bursts of these events, stagger ingestion jobs or temporarily pause new submissions so YouTube rate limits can cool down.
- Configure residential proxies or authenticated HTTP/S proxies via the `YOUTUBE_TRANSCRIPT_PROXY_URL` / `YOUTUBE_TRANSCRIPT_PROXY_HTTPS_URL` env vars to rotate IPs automatically.
- Provide browser cookies (e.g., `VISITOR_INFO1_LIVE`, `CONSENT`) via `YOUTUBE_TRANSCRIPT_COOKIES` or a `YOUTUBE_TRANSCRIPT_COOKIES_FILE` path so requests look like a real user session.
- When proxy rotation is unavailable, set `USE_WHISPER=true` to lean on the Whisper fallback for long-running jobs, accepting the higher latency/cost.

### Residential Proxy Mitigation (Decodo Site Unblocker example)
- Residential/rotating proxies are the most reliable way to keep `youtube_transcript_api` working from cloud networks. They mask Lambda’s datacenter IPs and auto-rotate when a request is blocked.
- For Lambda, extend `infra/terraform/terraform.tfvars` so the environment variables are injected automatically.
- After updating the env overrides, run `./scripts/redeploy_lambda.sh` to bake the proxy settings into Lambda, then rerun `scripts/sample_test_case_lambda.py` (or `scripts/test_transcript_proxy.py`) to verify ingestion completes.
- After updating the env overrides, run `./scripts/redeploy_lambda.sh` to bake the proxy settings into Lambda, then rerun `scripts/sample_test_case_lambda.py` (or `scripts/test_transcript_proxy.py`) to verify ingestion completes.
- Keep `USE_WHISPER` disabled to avoid large Lambda packages; the proxy path keeps the footprint small while still working around YouTube bans. If the provider mandates custom headers or SSL pinning, update the `YOUTUBE_TRANSCRIPT_PROXY_HEADERS` env var or trust store accordingly—the transcript service now injects those headers into the `youtube_transcript_api` HTTP client.
- Rotate all secret sources (`.env`, GitHub Actions / HF Spaces secrets, AWS Secrets Manager, Terraform vars) whenever proxy credentials change so that every deployment target stays in sync. After editing Terraform vars, re-run `terraform apply` to push the new environment overrides into Lambda.
- Validation checklist for any proxy change:
   1. Export the env vars locally and run `python3 scripts/test_transcript_proxy.py` to verify the unblocker responds (set `YOUTUBE_TRANSCRIPT_VERIFY_SSL=true` if the provider offers a trusted CA bundle).
   2. Deploy via Terraform / redeploy script so Lambda receives the rotated secrets.
   3. Tail `/aws/lambda/chaptly-api` logs for `get_transcript_with_api` warnings or `IpBlocked`/`RequestBlocked` signals to confirm no regressions.

### Proxy Cost & Safeguards
- Most pay-as-you-go residential pools bill per GB (commonly $8–$15/GB). Transcript fetches are lightweight (~150–300 KB/request), so steady-state costs stay low if you cache aggressively.
- Costs spike when:
   - You hammer retries (5–10 per request multiplies bandwidth).
   - A malicious user floods ingestion via the public HF Space.
   - You forget to cache transcripts and re-fetch the same video repeatedly.
- Mitigation checklist:
   - Cache transcripts/artifacts in S3/Dynamo keyed by `video_id` so subsequent requests never hit YouTube.
   - Enforce per-user token-bucket throttles plus a global daily cap before proxy calls to stop trolls from draining spend.
   - Detect proxy spend by approximating bytes transferred per transcript fetch; record totals in DynamoDB and stop once the monthly budget is reached.
   - Enable provider-side alerts/limits in the Decodo dashboard; combine with your own “hard stop” logic for defense in depth.
   - Back off with jitter (2–3 retries max) and require the proxy to rotate IPs each retry; never loop indefinitely.
   - Prioritize fetching official transcripts only once; fall back to Whisper only where legally permitted and offline (not in Lambda) to keep packages lean.

## Local Development & Setup
1. Install FFmpeg (see instructions below) if you plan to run Whisper.
2. `pip install -r requirements.txt` (CPU-friendly stack).
3. Export creds: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION=ap-southeast-1`, `GOOGLE_API_KEY`, `API_KEYS`, `ALLOWED_ORIGINS`.
4. Launch locally: `uvicorn src.api.main:app --host 0.0.0.0 --port 8000 --reload`.

### FFmpeg Installation (for optional Whisper transcription)
- **macOS**
   ```bash
   /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
   echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
   eval "$(/opt/homebrew/bin/brew shellenv)"
   brew install ffmpeg
   ffmpeg -version
   ```
- **Windows** – download from [ffmpeg.org](https://ffmpeg.org/download.html#build-windows), extract to `C:\ffmpeg`, add `C:\ffmpeg\bin` to PATH, verify with `ffmpeg -version`.
- **Linux**
   ```bash
   sudo apt update && sudo apt install ffmpeg     # Debian/Ubuntu
   sudo yum install ffmpeg                        # RHEL/CentOS
   ffmpeg -version
   ```


### Environment Variables (partial list)
| Variable | Purpose | Default |
|----------|---------|---------|
| `S3_BUCKET` | Artifact bucket name. | `chaptly-rag` |
| `S3_PREFIX` | Optional artifact folder prefix. | `` |
| `EMBEDDING_MODEL` | Embedding model id. | `models/text-embedding-004` |
| `GEMINI_MODEL` | Generation model. | `gemini-1.5-flash` |
| `MAX_REQUESTS_PER_MINUTE` | Rate-limit ceiling per key. | `60` |


## API Reference & Usage
| Method & Path | Description | Request Schema | Response Schema |
|---------------|-------------|----------------|-----------------|
| `POST /videos/process` | Queue ingestion for a YouTube URL. | Query param `url` (YouTube link). | `ProcessAccepted` (job id + video id). |
| `GET /videos/process/{job_id}` | Poll ingestion status. | Path `job_id`. | `ProcessStatus` (state, stats, message). |
| `GET /videos/{video_id}/bookmarks` | Return inferred sections. | Query `min_sections`, `max_sections`. | List[`Bookmark`]. |
| `POST /videos/{video_id}/qa` | Ask a grounded question. | `QARequest` (`query`, `limit`). | `QAResponse` (answer + sources). |
| `GET /videos/{video_id}/summary` | Summarize transcript. | Query `max_words`. | `SummaryResponse`. |
| `GET /videos/{video_id}/quiz` | Build quiz items. | Query `num_questions`, `style`. | List[`QuizItem`]. |
| `POST /search` | Semantic search across cached chunks. | `SearchRequest` (`query`, `video_id`, `limit`). | List[`SearchResult`]. |
| `GET /health` | Health probe. | None. | `{ "status": "ok" }`. |

### Sample Requests
```bash
# 1) Kick off ingestion (requires approved channel)
curl -X POST "${API_URL}/videos/process?url=https://www.youtube.com/watch?v=GOgA8JGUiwI" \
   -H "X-API-Key: ${CHAPTLY_API_KEY}"

# 2) Poll status
curl -X GET "${API_URL}/videos/process/${JOB_ID}" \
   -H "X-API-Key: ${CHAPTLY_API_KEY}"

# 3) Ask a question once artifacts exist
curl -X POST "${API_URL}/videos/GOgA8JGUiwI/qa" \
   -H "Content-Type: application/json" \
   -H "X-API-Key: ${CHAPTLY_API_KEY}" \
   -d '{"query":"What is dynamic programming?","limit":5}'

# 4) Semantic search
curl -X POST "${API_URL}/search" \
   -H "Content-Type: application/json" \
   -H "X-API-Key: ${CHAPTLY_API_KEY}" \
   -d '{"query":"bellman equation","video_id":"GOgA8JGUiwI","limit":4}'
```

Responses adhere to the Pydantic schemas in `src/api/models/schemas.py`, so you can auto-generate clients via FastAPI's OpenAPI doc (`${API_URL}/openapi.json`).

## Improvement Opportunities
1. Run embeddings on a managed vector DB (Qdrant/PG Vector) for cross-video search.
2. Add multilingual transcript normalization + translation flows.
3. Ship a background refresh job to detect YouTube transcript updates.
4. Expose metrics dashboards (CloudWatch dashboards, Honeycomb traces) for latency tracking.
5. Introduce signed client payloads using the HMAC secret for zero-trust deployments.

## References
- [Terms of Use](./terms-of-use.md)
- [Terraform Infrastructure Guide](./infra/terraform/README.md)
- [Deployment script](./scripts/package_lambda.sh)
- [API routers](./src/api/routers/)
- [Services & pipelines](./src/services/)

---

© Chaptly — see [LICENSE](./LICENSE) for usage terms. Contributions welcome via pull requests.
