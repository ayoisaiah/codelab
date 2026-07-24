summary: Learn how to instrument a Node.js application with OpenTelemetry. You'll go from zero visibility to finding two real performance bugs using distributed traces — without changing application code.
id: otel-workshop
categories: observability, opentelemetry, nodejs
environments: Web
status: Published
authors: Ayooluwa Isaiah

# Monitoring APIs with OpenTelemetry

## Introduction
Duration: 02:00

Distributed tracing captures the complete journey of a request as it moves through every service it touches. Each operation is recorded as a **span**, including its timing, status, and relevant contextual attributes.

OpenTelemetry is the standard framework for producing this data. Its Node.js SDK lets you instrument your application automatically — and start collecting traces with minimal setup.

In this codelab, you'll instrument a two-service Node.js application from scratch. You'll begin with auto-instrumentation to establish baseline visibility, then refine it with environment variables and SDK configuration. Finally, you'll add manual spans to capture business logic that auto-instrumentation cannot observe.

As you progress, you'll use the collected traces to uncover two real performance problems that are invisible in logs.

### What you'll learn

- How OpenTelemetry auto-instrumentation works
- How to set up the OTel Collector and Jaeger as a trace backend
- How to add manual spans and enrich existing spans with business context
- How to read trace waterfalls to find latency problems
- How to spot an N+1 query and a phantom cache miss using traces alone

### What you'll need

- Docker and Docker Compose
- Node.js 24+ and npm
- A terminal and a browser
- Basic familiarity with Express and REST APIs

## Set up the demo application
Duration: 05:00

The application you'll instrument is a URL shortener called **Snip**. It consists of two Node.js services backed by PostgreSQL and Redis.

**Shortener** is the public-facing service. It accepts URLs from users, fetches metadata (title and description) from the target page, stores everything in PostgreSQL, caches the mapping in Redis, and serves a web UI. When a user visits a short URL, this service proxies the request to the Redirector.

**Redirector** resolves short codes to original URLs. It checks Redis first, falls back to PostgreSQL on a cache miss, records visit analytics, and returns the original URL to the Shortener.

![Architecture diagram: Shortener and Redirector services backed by PostgreSQL and Redis](img/arch.png)

This architecture means a single redirect request flows through both services, hitting Redis, PostgreSQL, and an external API along the way — exactly the kind of multi-hop, multi-dependency flow where distributed tracing earns its keep.

### Clone and start the services

```bash
git clone https://github.com/ayoisaiah/nodejs-tracing-starter.git
cd nodejs-tracing-starter
mv .env.example .env
docker compose up -d --build
```

Once all containers are healthy, open `http://localhost:3000` in your browser to see the Snip UI.

![Snip UI with the URL input form and empty recent URLs table](img/snip-ui-empty.png)

Paste a URL like `https://opentelemetry.io` and click **Shorten**. The app fetches the page title, generates a short code, and displays the short URL.

![Snip UI after shortening a URL, showing the short URL and page title](img/snip-ui-shortened.png)

Click the short code in the **Recent URLs** table to trigger a redirect. The Shortener proxies the request to the Redirector, which resolves the short code and sends back the original URL.

<aside class="positive">
Everything works — but you have zero visibility into how these services interact to produce that result. If a redirect takes 5 seconds instead of 50ms, you can't tell which operation in the chain caused the delay. You'll fix that in the next step.
</aside>

## Enable auto-instrumentation
Duration: 08:00

Before writing any tracing code, it's worth understanding how far OpenTelemetry's auto-instrumentation can take you — for most Node.js applications, it's surprisingly far.

OpenTelemetry provides auto-instrumentation libraries for Express, `pg`, `redis`, `undici` (Node's built-in `fetch`), and many others. When loaded before your application code, these libraries monkey-patch the modules they target to automatically create spans. Every inbound HTTP request, every database query, every cache command, and every outbound `fetch()` call gets traced without writing a single line of instrumentation code.

Auto-instrumentation also handles context propagation across service boundaries. When the Shortener calls `fetch()` to reach the Redirector, the instrumentation automatically injects a `traceparent` header into the outbound request. On the Redirector side, the SDK extracts this header and continues the same trace. The result is a single distributed trace spanning both services — with no manual wiring required.

### Install the packages

```bash
npm install @opentelemetry/api \
  @opentelemetry/auto-instrumentations-node
```

- `@opentelemetry/api` defines the tracing interfaces (creating spans, setting attributes, propagating context).
- `@opentelemetry/auto-instrumentations-node` bundles the SDK and instrumentation for popular Node.js libraries.

### Verify with the console exporter

Add the following to your `.env` file:

```text
OTEL_TRACES_EXPORTER=console
OTEL_METRICS_EXPORTER=none
OTEL_LOGS_EXPORTER=none
NODE_OPTIONS=--import @opentelemetry/auto-instrumentations-node/register
```

Then reference them in `docker-compose.yml` for both services. Since each service needs its own `OTEL_SERVICE_NAME`, that one stays inline:

```yaml
services:
  shortener:
    environment:
      OTEL_SERVICE_NAME: shortener
      OTEL_TRACES_EXPORTER: ${OTEL_TRACES_EXPORTER}
      OTEL_METRICS_EXPORTER: ${OTEL_METRICS_EXPORTER}
      OTEL_LOGS_EXPORTER: ${OTEL_LOGS_EXPORTER}
      NODE_OPTIONS: ${NODE_OPTIONS}
  redirector:
    environment:
      OTEL_SERVICE_NAME: redirector
      OTEL_TRACES_EXPORTER: ${OTEL_TRACES_EXPORTER}
      OTEL_METRICS_EXPORTER: ${OTEL_METRICS_EXPORTER}
      OTEL_LOGS_EXPORTER: ${OTEL_LOGS_EXPORTER}
      NODE_OPTIONS: ${NODE_OPTIONS}
```

`NODE_OPTIONS` tells Node.js to load the auto-instrumentation module before your application code runs. Setting `OTEL_TRACES_EXPORTER` to `console` prints every completed span to stdout.

Rebuild and restart:

```bash
docker compose up -d --build
```

Shorten a URL and follow the redirect, then check the logs:

```bash
docker compose logs shortener redirector
```

You should see span objects printed to the terminal — each with a `name`, `traceId`, `duration`, and `attributes` field. If they're showing up, auto-instrumentation is working.

<aside class="positive">
Check that `service.name` in each span matches `shortener` or `redirector`. Without it, your telemetry lands under `unknown_service:node` — the last thing you want at 2am.
</aside>

<aside class="negative">
You may notice spans using deprecated attribute names like `http.method` instead of `http.request.method`. Newer instrumentation libraries are migrating to the updated semantic conventions. This doesn't affect how tracing works, but it matters when querying spans in your backend.
</aside>

## Set up the OTel Collector and Jaeger
Duration: 07:00

Console output confirmed that spans are being created. Now you need to send them to a tracing backend so you can explore traces visually.

### Create the Collector configuration

Create `otelcol.yaml` at your project root:

```yaml
receivers:
  otlp:
    protocols:
      http:
        endpoint: 0.0.0.0:4318

exporters:
  otlp/jaeger:
    endpoint: jaeger:4317
    tls:
      insecure: true
  debug:
    verbosity: basic

service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [otlp/jaeger, debug]
```

The Collector listens for OTLP over HTTP on port 4318 and exports to Jaeger over gRPC. The `debug` exporter prints a summary to stdout — useful for verifying spans are flowing through the pipeline.

### Add the Collector and Jaeger to Docker Compose

```yaml
services:
  collector:
    image: otel/opentelemetry-collector-contrib:0.148.0
    volumes:
      - ./otelcol.yaml:/etc/otelcol-contrib/config.yaml
    ports:
      - 4318:4318
    networks:
      - app

  jaeger:
    image: jaegertracing/jaeger:2.16.0
    ports:
      - 16686:16686
      - 4317:4317
    networks:
      - app
```

### Switch the exporter from console to OTLP

Update `.env`:

```text
OTEL_TRACES_EXPORTER=otlp
OTEL_EXPORTER_OTLP_ENDPOINT=http://collector:4318
```

Reference the new variable in `docker-compose.yml` for both services, and add a dependency on the Collector:

```yaml
services:
  shortener:
    environment:
      OTEL_EXPORTER_OTLP_ENDPOINT: ${OTEL_EXPORTER_OTLP_ENDPOINT}
    depends_on:
      collector:
        condition: service_started
  redirector:
    environment:
      OTEL_EXPORTER_OTLP_ENDPOINT: ${OTEL_EXPORTER_OTLP_ENDPOINT}
    depends_on:
      collector:
        condition: service_started
```

Rebuild and restart everything:

```bash
docker compose up -d --build
```

Shorten some URLs and visit them, then open Jaeger at `http://localhost:16686`. Select the **shortener** service and click **Find Traces**.

![Jaeger trace list showing multiple traces for the shortener service](img/jaeger-trace-list.png)

Click on the trace for a redirect request. You should see a distributed trace spanning both services.

![Jaeger trace detail showing 22 spans across the Shortener and Redirector services](img/jaeger-trace-detail.png)

The root span is the Shortener's GET request. Inside the Redirector's handler you can see the full resolution sequence: a Redis cache lookup, a PostgreSQL fallback, a Redis SET to re-populate the cache, an outbound call to ip-api.com for geolocation, and a PostgreSQL INSERT to record the visit.

**All of this happened without writing a single line of tracing code.** The auto-instrumentation libraries handled span creation, context propagation, and attribute population for every HTTP request, database query, and Redis command.

## Customize the auto-instrumentation
Duration: 06:00

The default auto-instrumentation gives you broad coverage, but two things stand out that are worth tuning.

### Filter out noisy spans

The trace includes spans for low-level operations like `dns.lookup`, `tcp.connect`, and `pg-pool.connect`. For most debugging workflows, these add clutter without diagnostic value.

The `OTEL_NODE_ENABLED_INSTRUMENTATIONS` variable lets you whitelist only the instrumentations you want. Add it to `.env`:

```text
OTEL_NODE_ENABLED_INSTRUMENTATIONS=http,express,pg,redis,undici,router
```

Reference it in `docker-compose.yml` for both services:

```yaml
OTEL_NODE_ENABLED_INSTRUMENTATIONS: ${OTEL_NODE_ENABLED_INSTRUMENTATIONS}
```

After restarting, traces in Jaeger will be significantly cleaner — showing only HTTP, Express, PostgreSQL, Redis, and outbound fetch operations.

![Jaeger trace after filtering: clean waterfall showing only meaningful spans](img/jaeger-filtered-spans.png)

If you prefer a blocklist instead, use `OTEL_NODE_DISABLED_INSTRUMENTATIONS`.

### Customize span names

Some spans show only "GET" or "POST" without indicating which endpoint was called. Fixing this requires programmatic SDK configuration.

Install the additional packages:

```bash
npm install @opentelemetry/sdk-node \
  @opentelemetry/exporter-trace-otlp-proto
```

Create `lib/otel.js`:

```javascript
import { getNodeAutoInstrumentations } from "@opentelemetry/auto-instrumentations-node";
import { OTLPTraceExporter } from "@opentelemetry/exporter-trace-otlp-proto";
import { NodeSDK } from "@opentelemetry/sdk-node";

const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter(),
  instrumentations: [
    getNodeAutoInstrumentations({
      "@opentelemetry/instrumentation-undici": {
        requestHook(span, request) {
          const url = new URL(request.origin + request.path);
          span.updateName(`${request.method} ${url.host}${url.pathname}`);
        },
      },
      "@opentelemetry/instrumentation-http": {
        requestHook(span, request) {
          span.updateName(`${request.method} ${request.path}`);
        },
      },
    }),
  ],
});

sdk.start();
process.on("SIGTERM", () => sdk.shutdown());
```

The `http` hook renames inbound server spans from a bare `GET` to something like `GET /resolve/BpW8ZVnQ`. The `undici` hook renames outbound `fetch()` spans to include the target host and path.

<aside class="positive">
The `SIGTERM` handler ensures the SDK flushes any buffered spans before the process exits. Without it, spans still in the export buffer when the process terminates are silently lost.
</aside>

Update `package.json` to use `--import`:

```json
{
  "scripts": {
    "shortener": "node --import ./lib/otel.js shortener/index.js",
    "redirector": "node --import ./lib/otel.js redirector/index.js"
  }
}
```

Remove the `NODE_OPTIONS` line from `.env` — it's been superseded by the `--import` flag. After restarting, both inbound and outbound HTTP spans in Jaeger will carry descriptive names.

![Jaeger trace with descriptive span names after customization](img/jaeger-custom-names.png)

## Add manual spans for business logic
Duration: 08:00

Auto-instrumentation combined with the customizations from the previous step already gives you substantial visibility. But it falls short inside your business logic.

The metadata extraction in the Shortener, the HTML parsing, the visit recording — these are all invisible in the current traces because no library boundary exists for the auto-instrumentation to hook into. Manual instrumentation fills that gap.

### Instrument the metadata extraction function

The `extractMetadata()` function in `shortener/metadata.js` fetches a target URL, checks the response, and parses the HTML. Several things can go wrong along the way. None of these failures surface in the auto-instrumented trace because they happen inside application logic.

Replace `shortener/metadata.js` with the instrumented version:

```javascript
import { SpanStatusCode, trace } from "@opentelemetry/api";
import * as cheerio from "cheerio";
import { logger } from "../lib/logger.js";

const tracer = trace.getTracer("shortener.metadata");

export async function extractMetadata(url) {
  return tracer.startActiveSpan("extract-metadata", async (span) => {
    span.setAttribute("url.full", url);
    try {
      const response = await fetch(url, {
        headers: { "User-Agent": "Shortener/1.0" },
        signal: AbortSignal.timeout(5000),
      });

      const contentType = response.headers.get("content-type") || "";
      if (!response.ok || !contentType.includes("text/html")) {
        throw new Error(`Unusable response: ${response.status} ${contentType}`);
      }

      const html = await response.text();
      span.setAttribute("metadata.html_bytes", html.length);

      const $ = cheerio.load(html);
      const title =
        $('meta[property="og:title"]').attr("content") ||
        $("title").first().text().trim() ||
        null;
      const description =
        $('meta[property="og:description"]').attr("content") ||
        $('meta[name="description"]').attr("content") ||
        null;

      span.setAttribute("metadata.has_title", title !== null);
      span.setAttribute("metadata.has_description", description !== null);
      return { title, description };
    } catch (err) {
      span.setStatus({ code: SpanStatusCode.ERROR, message: err.message });
      logger.error({ err }, "Failed to extract metadata");
      return { title: null, description: null };
    } finally {
      span.end();
    }
  });
}
```

A few things to note:

- `trace.getTracer("shortener.metadata")` obtains a tracer scoped to this component. `OTEL_SERVICE_NAME` identifies *which service*, the tracer name identifies *which component within that service*.
- `tracer.startActiveSpan()` makes the new span the active span. Any spans created inside the callback — including the auto-instrumented `fetch()` span — automatically become children.
- The `catch` block calls `span.setStatus()` to mark the span as failed in the trace UI.
- The `finally` block guarantees `span.end()` is called exactly once, ensuring the span is always exported.

<aside class="negative">
Do not call `span.end()` on spans owned by auto-instrumentation. The instrumentation ends them when the request completes. Calling `span.end()` yourself closes the span prematurely, before the handler finishes.
</aside>

After rebuilding, open the trace for `POST /api/shorten` in Jaeger. You should see the `extract-metadata` span nested under the HTTP server span.

![Jaeger trace showing the new extract-metadata span nested under the HTTP server span](img/jaeger-extract-metadata.png)

Expand the **Tags** panel to see the `metadata.has_title`, `metadata.has_description`, and `metadata.html_bytes` attributes.

![Jaeger span tags showing metadata attributes on the extract-metadata span](img/jaeger-extract-metadata-tags.png)

### Enrich existing auto-instrumented spans

You can attach your own attributes to auto-instrumented spans to make them searchable by dimensions that matter to your application.

In the Shortener's redirect proxy (`GET /:code`), add the short code and resolved URL:

```javascript
import { trace } from "@opentelemetry/api";

router.get("/:code", async (req, res) => {
  const { code } = req.params;
  try {
    const response = await fetch(`${REDIRECTOR_URL}/resolve/${code}`, {
      signal: AbortSignal.timeout(5000),
      redirect: "manual",
    });
    const body = await response.json();
    if (!response.ok) return res.status(response.status).json(body);

    const span = trace.getActiveSpan();
    if (span) {
      span.setAttribute("shortener.short_code", code);
      span.setAttribute("shortener.original_url", body.original_url);
    }
    res.redirect(302, body.original_url);
  } catch (err) {
    res.status(502).json({ error: "Redirector service unavailable" });
  }
});
```

![Jaeger span showing shortener.short_code and shortener.original_url attributes on the Shortener span](img/jaeger-span-attrs-shortener.png)

In the Redirector's `resolve/:code` handler, record whether the cache was hit:

```javascript
import { trace } from "@opentelemetry/api";

router.get("/resolve/:code", async (req, res) => {
  const { code } = req.params;
  try {
    const cached = await redis.get(`urls:${code}`);
    const span = trace.getActiveSpan();
    if (span) {
      span.setAttribute("shortener.short_code", code);
      span.setAttribute("shortener.cache_hit", !!cached);
    }
    // ...rest of handler
  } catch (err) {
    res.status(500).json({ error: "Internal server error" });
  }
});
```

![Jaeger span showing shortener.cache_hit attribute on the Redirector span](img/jaeger-span-attrs-redirector.png)

<aside class="positive">
The `if (span)` guard is a defensive pattern worth adopting. `getActiveSpan()` returns `undefined` if the SDK isn't initialized, so the guard prevents your application from crashing when tracing is absent or disabled.
</aside>

The `shortener.cache_hit` attribute is particularly valuable — it tells you immediately whether a redirect was served from Redis or fell back to PostgreSQL. You'll see exactly why in the next step.

## Debug your services with traces
Duration: 10:00

The instrumentation you've built gives you visibility into the structure and timing of every request. Now put that visibility to work.

The demo application has two subtle performance problems baked in that are **invisible from the outside** — no errors, no crashes, everything functions correctly. Tracing is how you'll find them.

### Bug 1: The N+1 query

Open `http://localhost:3000` to load the Snip UI. It loads normally and shows all your shortened URLs. Nothing looks wrong.

Open Jaeger, find the trace for the `GET /api/urls` request, and look at the span timeline.

![Jaeger trace for GET /api/urls showing a cascade of sequential PostgreSQL spans — the N+1 query pattern](img/jaeger-n1-query.png)

Instead of one or two database spans, there's a cascade of `pg-pool.connect`, `pg.connect`, and `pg.query:SELECT` spans repeating in sequence. Each URL in the list triggers its own connection acquisition and count query.

Here's the code causing it:

```javascript
// shortener/routes.js
router.get("/api/urls", async (_req, res) => {
  const result = await db.query(
    "SELECT short_code, original_url, title, description, created_at FROM urls ORDER BY created_at DESC LIMIT 20",
  );

  // Fetches visit count for each URL individually
  const rows = await Promise.all(
    result.rows.map(async (row) => {
      const visits = await db.query(
        "SELECT COUNT(*) FROM visits WHERE short_code = $1",
        [row.short_code],
      );
      return { ...row, visit_count: parseInt(visits.rows[0].count, 10) };
    }),
  );

  res.json(rows);
});
```

In your PostgreSQL logs, every query looks successful. But the trace tells a different story: the waterfall of sequential queries adds up to a total latency that is the *sum* of all of them, not the cost of any single one.

**The fix** — collapse the N+1 into a single `LEFT JOIN`:

```javascript
router.get("/api/urls", async (_req, res) => {
  const result = await db.query(
    `SELECT u.short_code, u.original_url,
            u.title, u.description,
            u.created_at,
            COUNT(v.id)::int AS visit_count
     FROM urls u
     LEFT JOIN visits v ON u.short_code = v.short_code
     GROUP BY u.id
     ORDER BY u.created_at DESC
     LIMIT 20`,
  );
  res.json(result.rows);
});
```

After applying the fix and restarting, the same request produces a trace with a single PostgreSQL span and the total duration drops accordingly.

![Jaeger trace for GET /api/urls after the fix — a single PostgreSQL span replaces the cascade](img/jaeger-n1-fixed.png)

### Bug 2: The phantom cache miss

Create a short URL, then visit it immediately. Open Jaeger and look at the `shortener.cache_hit` attribute on the Redirector's `request handler - /resolve/:code` span.

![Jaeger span showing shortener.cache_hit: false on the first redirect after URL creation](img/jaeger-cache-miss.png)

It should be `true` — you just created this URL and the Shortener cached it in Redis. But it reads `false`. Visit the same URL a second time and it reads `true`.

So the cache works on subsequent requests, but the very first redirect after creation always misses. Something is wrong between the write and the first read.

Check the Redis key the Shortener writes on URL creation:

```javascript
// shortener/routes.js
await redis.set(`url:${shortCode}`, url, { EX: 86400 });
```

Now check what the Redirector reads:

```javascript
// redirector/routes.js
const cached = await redis.get(`urls:${code}`);
```

Spot it? `url:` vs `urls:`. On the first redirect, the Redirector reads from a key that was never written, falls back to PostgreSQL, then writes its own entry under the `urls:` prefix. Subsequent redirects hit *that* entry and appear to work fine — which is exactly why this bug is so easy to miss.

**The fix** — use the correct key format in the Redirector:

```javascript
// redirector/routes.js
const cached = await redis.get(`url:${code}`);
// ...
await redis.set(`url:${code}`, originalUrl, { EX: 86400 });
```

After rebuilding, create a new short URL and visit it immediately. The first visit should now show `shortener.cache_hit: true`, and the PostgreSQL `SELECT` span disappears.

![Jaeger span showing shortener.cache_hit: true after the fix](img/jaeger-cache-hit.png)

<aside class="positive">
**What logs alone would have told you:** the N+1 query logged a series of successful database operations — nothing looked wrong. The cache miss logged nothing at all, because from the application's perspective it wasn't an error. Tracing revealed both problems by showing how operations relate to each other in time and across services, not just whether each individual operation succeeded.
</aside>

## What's next
Duration: 02:00

You started this codelab with a working application and zero observability. By the end, you've:

- Added auto-instrumentation that captures every HTTP request, database query, and cache command across two services
- Customized span names to make traces readable at a glance
- Created manual spans that expose business logic the auto-instrumentation can't reach
- Added custom attributes that make spans queryable by business dimensions
- Used the resulting traces to find two performance bugs that no amount of logging would have revealed

Everything you've built here uses open standards. The traces export over OTLP, the attributes follow OpenTelemetry semantic conventions, and the SDK configuration is portable across backends.

### Sampling

This codelab samples 100% of traces, which is appropriate for development but not for most production environments. OpenTelemetry supports several [sampling strategies](https://opentelemetry.io/docs/concepts/sampling/) configurable through the SDK and Collector. Getting sampling right is essential for controlling costs while still capturing the traces that matter.

### Further reading

- [Configuring OTel with environment variables](https://www.dash0.com/guides/opentelemetry-environment-variables) — the full reference for `OTEL_*` vars
- [Building telemetry pipelines with the OTel Collector](https://www.dash0.com/guides/opentelemetry-collector) — receivers, processors, exporters
- [Bridging Python's logging module to OTel](https://www.dash0.com/guides/opentelemetry-logging-python) — log-trace correlation
- [OpenTelemetry semantic conventions](https://opentelemetry.io/docs/specs/semconv/) — the shared attribute vocabulary

### Send traces to a production backend

Switching from Jaeger to any OTLP-compatible backend is a Collector config change — no application code changes required.

```yaml
exporters:
  otlp/my-backend:
    endpoint: https://your-backend-endpoint
    headers:
      Authorization: "Bearer ${AUTH_TOKEN}"

service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [otlp/my-backend]
```
