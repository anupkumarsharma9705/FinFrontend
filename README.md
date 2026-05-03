# fintech-frontend

A minimal static HTML demo interface for the Adaptive Fintech API Security Gateway. Lets recruiters, reviewers, and interviewers interact with all gateway features through a browser without any setup.

---

## Table of Contents

1. [Purpose](#purpose)
2. [Tech Stack](#tech-stack)
3. [File Structure](#file-structure)
4. [What Each Section Demonstrates](#what-each-section-demonstrates)
5. [How to Run](#how-to-run)
6. [Configuration](#configuration)
7. [Running with Docker and Nginx](#running-with-docker-and-nginx)
8. [Design Decisions](#design-decisions)
9. [Security Notes](#security-notes)

---

## Purpose

This frontend exists for one reason — to make the gateway's features visible and interactive without requiring technical setup from the person reviewing it.

A recruiter or interviewer should be able to:

1. Open a browser
2. Click a button
3. See the gateway intercept, score, and make a decision on a request in real time

Every section maps directly to a real engineering concept implemented in the gateway:

| Section | Concept Demonstrated |
|---|---|
| System Health | Health monitoring, dependency checking, graceful degradation |
| Normal Request | Proxy layer, risk scoring at baseline, ALLOW decision |
| Payment + Idempotency | Duplicate payment prevention, three-state idempotency |
| SQL Injection | Attack detection, pre-backend blocking |
| Rate Limiting | Adaptive scoring, THROTTLE → BLOCK progression |
| Feedback API | False positive recovery, trust tier reset |

---

## Tech Stack

| Technology | Version | Purpose |
|---|---|---|
| HTML5 | — | Structure |
| CSS3 | — | Styling (dark theme, inline) |
| Vanilla JavaScript | ES2020 | API calls via `fetch()` |
| Nginx | Alpine | Static file server + API proxy (Docker) |

**Zero dependencies.** No React, no Vue, no npm, no build step. Open the file in a browser and it works.

---

## File Structure

```
frontend/
└── index.html          ← Entire frontend in one file

nginx.conf              ← Nginx config for Docker deployment
```

The entire frontend is a single self-contained HTML file. Styles are inline in `<style>` tags. JavaScript is inline in `<script>` tags. No external files, no CDN dependencies.

---

## What Each Section Demonstrates

### Section 1 — System Health

**Button:** Check Health

**Calls:** `GET /health`

**What you see:**
```json
{
  "redis": "UP",
  "mysql": "UP",
  "backend": "UP",
  "gateway": "UP",
  "overall": "HEALTHY"
}
```

**What it demonstrates:**
- Gateway monitors all its dependencies in real time
- If Redis goes down, response becomes `"overall": "DEGRADED"` with status 503
- Gateway never crashes — health check itself works even when dependencies fail

**To demonstrate degradation:** Run `docker stop fintech_redis` and click again.

---

### Section 2 — Normal Request (ALLOW)

**Input:** Account ID (default: ACC123)
**Button:** Get Balance

**Calls:** `GET /api/account/balance?accountId={accountId}`

**What you see:**
```json
{
  "accountId": "ACC123",
  "balance": 50000.0,
  "currency": "INR"
}
```

**What it demonstrates:**
- Gateway intercepts the request
- Risk score is calculated (10 points for balance endpoint, daytime)
- Score 10 < threshold 31 → ALLOW decision
- Request forwarded to backend, response returned

**Console shows:**
```
Risk Score: 10  |  Decision: ALLOW
```

---

### Section 3 — Payment + Idempotency

**Input:** Idempotency Key (default: PAY-DEMO-001)
**Buttons:** Send Payment | Send Again (duplicate)

**Calls:** `POST /api/payments/transfer` with `Idempotency-Key` header

**First click — what you see:**
```json
{
  "status": "SUCCESS",
  "transactionId": "TXN-1776059802163",
  ...
}
```

**Second click (same key) — what you see:**
```json
{
  "status": "SUCCESS",
  "transactionId": "TXN-1776059802163",   ← identical transaction ID
  ...
}
```

**What it demonstrates:**
- Both calls return identical `transactionId`
- Second call never reached the backend — response was served from Redis cache
- Payment was NOT processed twice
- Console shows: `IDEMPOTENT: Returning cached response for key: PAY-DEMO-001`

**To prove backend wasn't called twice:** Change the idempotency key and click — you'll get a new `transactionId` with different digits.

---

### Section 4 — SQL Injection Attack

**Button:** Send SQL Injection Attack (red)

**Calls:** `POST /api/payments/transfer` with malicious payload:
```json
{
  "fromAccountId": "ACC123' OR '1'='1",
  ...
}
```

**What you see:**
```json
{
  "error": "Malicious request detected",
  "threatType": "SQL_INJECTION"
}
```

**What it demonstrates:**
- Attack is detected in the request body before the backend ever sees it
- HTTP 400 Bad Request returned
- Attack type is identified and logged
- Gateway console shows: `ATTACK DETECTED: SQL_INJECTION`
- MySQL audit_logs records: `attack_type = SQL_INJECTION`, `decision = BLOCK`

**Why this matters:** If this request reached a real backend with an unsanitized SQL query, it could expose or destroy the entire database. The gateway stops it at the edge.

---

### Section 5 — Adaptive Rate Limiting

**Button:** Send 15 Rapid Requests

**Calls:** 15 sequential `GET /api/account/balance` requests in a loop

**What you see:**
```
Request 1:  HTTP 200
Request 2:  HTTP 200
...
Request 10: HTTP 200  (score climbs to ~20, THROTTLE starts)
Request 11: HTTP 200  (2 second delay — browser becomes slow)
...
Request 14: HTTP 200  (score at 50, still throttling)
Request 15: HTTP 429  (BLOCK — score exceeded threshold)
```

**What it demonstrates:**
- Risk score increases with each request
- THROTTLE phase: requests still succeed but with progressive delays
- BLOCK phase: requests rejected with HTTP 429
- Score is adaptive — it considers the endpoint type, frequency, and trust tier
- After 1 minute, the Redis sliding window resets and score returns to baseline

**To see trust tier degradation:** After getting blocked, wait 10 seconds and send one request. Score will be 30 instead of 10 — LOW trust adds +20 permanently until cooldown expires.

---

### Section 6 — Feedback API (False Positive Recovery)

**Input:** IP to unblock (default: 0:0:0:0:0:0:0:1 which is localhost IPv6)
**Button:** Report False Positive

**Calls:** `POST /feedback/false-positive`

**What you see:**
```json
{
  "status": "resolved",
  "ip": "0:0:0:0:0:0:0:1",
  "message": "IP trust tier reset to MEDIUM"
}
```

**What it demonstrates:**
- Ops teams can recover falsely blocked users without restarting the gateway
- Redis blacklist key is deleted
- Block count is reset
- Trust tier is restored to MEDIUM
- Trust history record is saved to MySQL: `BLACKLISTED → MEDIUM`
- After unblocking, the normal request section works again at score 10

**Best demo flow:**
1. Click "Send 15 Rapid Requests" until blocked (HTTP 429)
2. Try "Get Balance" — get 403 Access Denied
3. Click "Report False Positive"
4. Try "Get Balance" again — works at score 10

---

## How to Run

### Option 1 — Direct Browser (Simplest)

```bash
# Open the file directly
open frontend/index.html        # macOS
start frontend/index.html       # Windows
xdg-open frontend/index.html    # Linux
```

Make sure both `fintech-backend` (port 8080) and `fintech-gateway` (port 8081) are running locally first.

**Note:** Direct file access (`file://`) may trigger CORS restrictions in some browsers. If buttons do nothing, use Option 2 or Option 3.

### Option 2 — Simple HTTP Server

```bash
cd frontend

# Python 3
python3 -m http.server 3000

# Node.js
npx serve . -p 3000
```

Open: `http://localhost:3000`

### Option 3 — Docker with Nginx (Production)

```bash
# From project root
docker-compose up frontend
```

Open: `http://localhost`

Nginx proxies `/api/`, `/health`, `/ping`, and `/feedback/` to the gateway — no CORS issues.

---

## Configuration

The gateway URL is set at the top of the `<script>` section in `index.html`:

```javascript
const GATEWAY = 'http://localhost:8081';
```

Change this to your deployed gateway URL for cloud deployment:

```javascript
const GATEWAY = 'https://your-gateway.railway.app';
```

When running via Docker Compose with Nginx, the frontend calls `/api/...` which Nginx proxies to the gateway internally — no hardcoded URL needed in that case.

---

## Running with Docker and Nginx

The `nginx.conf` file configures Nginx to:
1. Serve `index.html` as the root page
2. Proxy API calls to the gateway container (no CORS)
3. Add security headers to all responses
4. Enable gzip compression

```nginx
# Proxy all API calls to gateway
location /api/ {
    proxy_pass http://fintech-gateway:8081;
}

location /health {
    proxy_pass http://fintech-gateway:8081/health;
}

location /feedback/ {
    proxy_pass http://fintech-gateway:8081;
}
```

When running via Docker Compose, the frontend container is on the same `fintech_network` as the gateway — they communicate by container name.

### Build and Run

```bash
docker-compose up frontend
```

Access at: `http://localhost`

---

## Design Decisions

### Why No Framework?

React, Vue, or Angular would add:
- npm install step
- Build step
- `node_modules` folder
- Webpack/Vite config

None of these add value for a demo page. Vanilla JS with `fetch()` achieves the same result in 150 lines.

Anyone reviewing the code can read it immediately — no framework knowledge required.

### Why Dark Theme?

Dark backgrounds with green and blue accents are standard in security tooling and monitoring dashboards. The visual style signals that this is a security/infrastructure tool, not a consumer product.

### Why Red Button for Attack?

The SQL injection button is styled red with `class="danger"` to make it visually clear this action is intentionally malicious — it reduces confusion for reviewers who might wonder if the button is broken when they see a 400 response.

### Why Sequential Requests for Rate Limit Demo?

```javascript
for (let i = 0; i < 15; i++) {
    const res = await fetch(...);
    results.push(`Request ${i+1}: HTTP ${res.status}`);
}
```

Using `await` in a loop sends requests one at a time, sequentially. This gives the rate limiter time to increment the counter between requests and makes the progression from 200 → throttle → 429 visible and predictable. Parallel requests would be less predictable in demo conditions.

### Why Update Results Incrementally?

```javascript
results.push(`Request ${i+1}: HTTP ${status}`);
document.getElementById('rateResponse').textContent = results.join('\n');
```

The response div is updated after every request rather than waiting for all 15 to complete. This lets the reviewer watch the status codes change in real time — demonstrating the adaptive nature of the scoring system as it climbs.

---

## Security Notes

### CORS

When running locally (`file://` or `localhost:3000`), the browser sends CORS preflight requests. The gateway's `GatewayConfig` includes a CORS configuration that allows all origins for development:

```java
registry.addMapping("/**").allowedOrigins("*").allowedMethods("*");
```

In production, restrict `allowedOrigins` to your specific domain.

### The Feedback Endpoint

The feedback button calls `POST /feedback/false-positive`. In the demo this is intentionally accessible — but this endpoint has no authentication. In production:
- Restrict to internal network only
- Or require admin JWT token
- Or remove from public-facing deployment entirely

### No Sensitive Data

The demo uses only simulated data — `ACC123`, `USER001`, `PAY-DEMO-001`. No real account numbers, no real money, no real personal information.
