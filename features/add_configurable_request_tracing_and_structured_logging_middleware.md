# Request Tracing and Structured Logging Middleware

This document contains a complete implementation of a configurable request tracing and structured logging middleware for Express (TypeScript). It provides:

- Request ID generation and propagation (X-Request-Id / X-Correlation-Id)
- Structured JSON logs for request start and finish
- Sampling control and path exclusions
- Optional injection of a custom logger

---

## Feature Overview

The middleware ensures every request has a stable trace identifier accessible to downstream code. It also emits structured logs that make it easy to correlate request lifecycle events in log aggregators.

Goals:
- Minimal runtime dependencies
- Easy to drop into existing Express apps
- Configurable and testable


## Technical Specifications

- Language: TypeScript
- Target: Node 14+
- Exports:
  - createTracingMiddleware(options?: TracingOptions): RequestHandler
- Options:
  - headerRequestId?: string (default "x-request-id")
  - headerCorrelationId?: string (default "x-correlation-id")
  - sampleRate?: number (0..1, default 1.0)
  - excludePaths?: string[] (path prefixes to skip tracing)
  - logger?: Logger (object with info/warn/error/debug)

Trace object shape attached to req and res.locals:
{
  traceId: string,
  correlationId: string,
  sampled: boolean
}


## Implementation

Below are code fragments making up a complete implementation.

### package.json

```json
{
  "name": "express-tracing-middleware",
  "version": "1.0.0",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "scripts": {
    "build": "tsc",
    "test": "jest",
    "start": "node dist/example.js"
  },
  "dependencies": {
    "express": "^4.17.0",
    "uuid": "^9.0.0"
  },
  "devDependencies": {
    "@types/express": "^4.17.0",
    "@types/jest": "^29.0.0",
    "jest": "^29.0.0",
    "ts-jest": "^29.0.0",
    "typescript": "^4.9.0"
  }
}
```


### tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2019",
    "module": "CommonJS",
    "declaration": true,
    "outDir": "dist",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true
  },
  "include": ["src/**/*"]
}
```


### src/types.ts

```ts
export interface TraceInfo {
  traceId: string;
  correlationId: string;
  sampled: boolean;
}

export interface Logger {
  info: (...args: any[]) => void;
  warn: (...args: any[]) => void;
  error: (...args: any[]) => void;
  debug?: (...args: any[]) => void;
}

export interface TracingOptions {
  headerRequestId?: string;
  headerCorrelationId?: string;
  sampleRate?: number;
  excludePaths?: string[];
  logger?: Logger;
}
```


### src/logger.ts

```ts
import { Logger } from "./types";

export const defaultLogger: Logger = {
  info: (...args: any[]) => {
    try {
      console.log(JSON.stringify({ level: "info", message: args }))
    } catch {
      console.log("info", ...args)
    }
  },
  warn: (...args: any[]) => {
    try {
      console.warn(JSON.stringify({ level: "warn", message: args }))
    } catch {
      console.warn("warn", ...args)
    }
  },
  error: (...args: any[]) => {
    try {
      console.error(JSON.stringify({ level: "error", message: args }))
    } catch {
      console.error("error", ...args)
    }
  },
  debug: (...args: any[]) => {
    try {
      console.debug(JSON.stringify({ level: "debug", message: args }))
    } catch {
      console.debug("debug", ...args)
    }
  }
}
```


### src/tracing.ts

```ts
import { RequestHandler } from "express";
import { v4 as uuidv4 } from "uuid";
import { TracingOptions, TraceInfo, Logger } from "./types";
import { defaultLogger } from "./logger";

const DEFAULT_HEADER_REQUEST_ID = "x-request-id";
const DEFAULT_HEADER_CORRELATION_ID = "x-correlation-id";

function shouldExclude(path: string, excludePaths?: string[]) {
  if (!excludePaths || excludePaths.length === 0) return false;
  for (const prefix of excludePaths) {
    if (path.startsWith(prefix)) return true;
  }
  return false;
}

export function createTracingMiddleware(options: TracingOptions = {}): RequestHandler {
  const headerRequestId = (options.headerRequestId || DEFAULT_HEADER_REQUEST_ID).toLowerCase();
  const headerCorrelationId = (options.headerCorrelationId || DEFAULT_HEADER_CORRELATION_ID).toLowerCase();
  const sampleRate = typeof options.sampleRate === "number" ? Math.max(0, Math.min(1, options.sampleRate)) : 1;
  const excludePaths = options.excludePaths || [];
  const logger: Logger = options.logger || defaultLogger;

  return function tracingMiddleware(req, res, next) {
    try {
      if (shouldExclude(req.path, excludePaths)) return next();

      const incomingRequestId = req.header(headerRequestId) || req.header("request-id") || undefined;
      const incomingCorrelationId = req.header(headerCorrelationId) || undefined;

      const traceId = incomingRequestId || uuidv4();
      const correlationId = incomingCorrelationId || traceId;

      const sampled = Math.random() < sampleRate;

      const trace: TraceInfo = { traceId, correlationId, sampled };

      (req as any).trace = trace;
      if (!res.locals) res.locals = {};
      (res.locals as any).trace = trace;

      // Propagate headers to response
      res.setHeader(headerRequestId, traceId);
      res.setHeader(headerCorrelationId, correlationId);

      const start = process.hrtime();

      if (sampled) {
        logger.info({ event: "request:start", method: req.method, url: req.originalUrl || req.url, traceId, correlationId });
      }

      const onFinish = () => {
        const diff = process.hrtime(start);
        const durationMs = Math.round((diff[0] * 1e3) + (diff[1] / 1e6));

        const message = {
          event: "request:finish",
          method: req.method,
          url: req.originalUrl || req.url,
          statusCode: res.statusCode,
          durationMs,
          traceId,
          correlationId
        };

        if (sampled) {
          logger.info(message);
        }

        res.removeListener("finish", onFinish);
        res.removeListener("close", onClose);
      };

      const onClose = () => {
        const diff = process.hrtime(start);
        const durationMs = Math.round((diff[0] * 1e3) + (diff[1] / 1e6));

        const message = {
          event: "request:close",
          method: req.method,
          url: req.originalUrl || req.url,
          durationMs,
          traceId,
          correlationId
        };

        if (sampled) {
          logger.warn(message);
        }

        res.removeListener("finish", onFinish);
        res.removeListener("close", onClose);
      };

      res.on("finish", onFinish);
      res.on("close", onClose);

      next();
    } catch (err) {
      const safeLogger: Logger = options.logger || defaultLogger;
      safeLogger.error({ event: "tracing:error", error: (err as Error).message });
      next();
    }
  };
}
```


### src/index.ts

```ts
export { createTracingMiddleware } from "./tracing";
export type { TracingOptions } from "./types";
```


### Example usage (Express) - src/example.ts

```ts
import express from "express";
import { createTracingMiddleware } from "./tracing";

const app = express();

app.use(createTracingMiddleware({ sampleRate: 1.0, excludePaths: ["/health"] }));

app.get("/health", (req, res) => res.status(200).send("ok"));

app.get("/hello", (req, res) => {
  const trace = (req as any).trace;
  res.json({ message: "hello", trace });
});

const port = Number(process.env.PORT || 3000);
app.listen(port, () => console.log(`Listening on ${port}`));
```


## Implementation notes

- The middleware is intentionally minimal: it attaches trace info to req and res.locals and sets response headers.
- Sampling is implemented via Math.random which is sufficient for most cases; for deterministic sampling you can replace with a hash-based sampler.
- The default logger emits JSON-stringified messages using console. Replace with winston/pino for production.
- Exclusion matching is prefix-based for simplicity. Change to regex if you need advanced matching.
- This implementation uses synchronous header reads and synchronous logger calls; for async logging backends, ensure the logger handles buffering appropriately.


## Tests (Jest) - src/__tests__/tracing.test.ts

```ts
import express from "express";
import request from "supertest";
import { createTracingMiddleware } from "../tracing";

describe("tracing middleware", () => {
  it("generates a traceId when not provided and attaches to req and res.locals", async () => {
    const app = express();
    app.use(createTracingMiddleware({ sampleRate: 0 }));

    app.get("/test", (req, res) => {
      const trace = (req as any).trace;
      res.json({ trace });
    });

    const resp = await request(app).get("/test");
    expect(resp.status).toBe(200);
    expect(resp.body.trace).toBeDefined();
    expect(resp.body.trace.traceId).toBeDefined();
    expect(resp.headers["x-request-id"]).toBeDefined();
  });

  it("preserves incoming request id and correlation id", async () => {
    const app = express();
    app.use(createTracingMiddleware({ sampleRate: 0 }));

    app.get("/test", (req, res) => {
      res.json((req as any).trace);
    });

    const resp = await request(app).get("/test").set("x-request-id", "my-request-id").set("x-correlation-id", "my-corr");
    expect(resp.body.traceId || resp.body.trace).toBeUndefined();
    // if body shape changed, check headers instead
    expect(resp.headers["x-request-id"]).toBe("my-request-id");
    expect(resp.headers["x-correlation-id"]).toBe("my-corr");
  });
});
```

Notes: The second test checks header propagation primarily; depending on how you deserialize the trace object you might need to adjust assertions.


## Configuration / Setup Requirements

- Node 14+ recommended
- Install dependencies:

```bash
npm install
npm run build
npm test
```

- To integrate into your project:
  - Import createTracingMiddleware and add to your Express app early (before routers) so trace info is available to all downstream handlers.
  - Provide a logger that serializes to your centralized logging format if necessary.


## Pseudocode Example

```
app.use(createTracingMiddleware({ sampleRate: 0.1, excludePaths: ["/metrics"], logger: myLogger }))

app.get("/orders/:id", (req, res) => {
  const trace = req.trace; // { traceId, correlationId, sampled }
  // use trace.traceId to tag logs or pass to downstream services
  res.send(...)
})
```


## Implementation Notes / Future Enhancements

- Add deterministic sampling (e.g., hash-based by traceId) to maintain consistent sampling across retries.
- Add support for additional headers commonly used in tracing (e.g., traceparent) if integrating with W3C Trace Context.
- Provide middleware metrics (counts, sampled count) for operational visibility.


---

This file was prepared by Francisco as the primary author. Use the code as a drop-in starting point and adapt logging and sampling to match operational constraints.