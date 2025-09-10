# Rate Limiter Middleware (by Francisco)

## Feature overview

This module provides a configurable rate-limiting middleware for Express applications. It supports:
- In-memory token bucket store (default)
- Optional Redis-backed store for distributed deployments
- Burst configuration and window-based refill
- Custom key generation (IP, user id, API key, etc.)
- Hooks for custom behavior when limit is reached

The implementation is written in TypeScript and is intended to be dropped into existing Express apps with minimal changes.

## Technical specifications

- Defaults:
  - windowMs: 60_000 (1 minute)
  - maxRequests: 60
  - burst: 10 (instant tokens above the steady rate)
- Token bucket semantics: each key has a capacity of `maxRequests + burst`. Tokens are refilled linearly per `windowMs`.
- Store interface (required methods):
  - async get(key: string): Promise<TokenRecord | null>
  - async set(key: string, record: TokenRecord): Promise<void>
  - async incr?(key: string): Promise<number>  // optional convenience for fixed-window stores
  - async cleanup?(): Promise<void> // optional

TokenRecord shape:
- tokens: number
- lastRefill: number (epoch ms)
- capacity: number

## Code (TypeScript)

```ts
// rateLimiter.ts
import { Request, Response, NextFunction } from 'express';

export type TokenRecord = {
  tokens: number;
  lastRefill: number; // epoch ms
  capacity: number;
};

export type Store = {
  get: (key: string) => Promise<TokenRecord | null>;
  set: (key: string, record: TokenRecord) => Promise<void>;
  cleanup?: () => Promise<void>;
};

export type RateLimiterOptions = {
  windowMs?: number;
  maxRequests?: number;
  burst?: number;
  keyGenerator?: (req: Request) => string;
  store?: Store;
  onLimitReached?: (req: Request, res: Response) => void;
};

const DEFAULTS = {
  windowMs: 60_000,
  maxRequests: 60,
  burst: 10,
};

// In-memory store implementation
export function createInMemoryStore(cleanupIntervalMs = 60_000): Store {
  const map = new Map<string, TokenRecord>();

  const cleanup = async () => {
    const now = Date.now();
    for (const [key, rec] of map.entries()) {
      // if a key hasn't been touched for 2x window, remove it
      if (now - rec.lastRefill > 2 * DEFAULTS.windowMs) {
        map.delete(key);
      }
    }
  };

  const interval = setInterval(cleanup, cleanupIntervalMs);
  // Node.js apps should clear interval on shutdown if desired

  return {
    get: async (key: string) => map.get(key) || null,
    set: async (key: string, record: TokenRecord) => map.set(key, record),
    cleanup: async () => {
      clearInterval(interval);
      await cleanup();
    },
  };
}

function refillTokens(record: TokenRecord, now: number, windowMs: number, ratePerMs: number) {
  const elapsed = Math.max(0, now - record.lastRefill);
  const refill = elapsed * ratePerMs;
  if (refill > 0) {
    record.tokens = Math.min(record.capacity, record.tokens + refill);
    record.lastRefill = now;
  }
}

export function rateLimiter(opts: RateLimiterOptions = {}) {
  const windowMs = opts.windowMs ?? DEFAULTS.windowMs;
  const maxRequests = opts.maxRequests ?? DEFAULTS.maxRequests;
  const burst = opts.burst ?? DEFAULTS.burst;
  const capacity = maxRequests + burst;
  const ratePerMs = maxRequests / windowMs; // tokens per ms to achieve steady rate
  const keyGenerator = opts.keyGenerator ?? ((req: Request) => req.ip || req.headers['x-forwarded-for'] as string || 'global');
  const store: Store = opts.store ?? createInMemoryStore();
  const onLimitReached = opts.onLimitReached;

  return async function middleware(req: Request, res: Response, next: NextFunction) {
    try {
      const key = keyGenerator(req);
      const now = Date.now();

      let record = await store.get(key);
      if (!record) {
        record = { tokens: capacity - 1, lastRefill: now, capacity };
        await store.set(key, record);
        // allow request
        res.setHeader('X-RateLimit-Limit', String(maxRequests));
        res.setHeader('X-RateLimit-Remaining', String(Math.floor(record.tokens)));
        return next();
      }

      // refill
      refillTokens(record, now, windowMs, ratePerMs);

      // check tokens
      if (record.tokens >= 1) {
        record.tokens = record.tokens - 1;
        await store.set(key, record);
        const remainingSteady = Math.max(0, Math.floor(record.tokens));
        res.setHeader('X-RateLimit-Limit', String(maxRequests));
        res.setHeader('X-RateLimit-Remaining', String(remainingSteady));
        return next();
      }

      // limit reached
      await store.set(key, record);
      const retryAfterSeconds = Math.ceil((1 - record.tokens) / ratePerMs / 1000);
      res.setHeader('Retry-After', String(retryAfterSeconds));
      res.status(429).json({
        message: 'Too many requests, please try again later.',
        limit: maxRequests,
        retryAfter: retryAfterSeconds,
      });
      if (onLimitReached) {
        try {
          onLimitReached(req, res);
        } catch (err) {
          // swallow errors from hook
        }
      }
    } catch (err) {
      // If the limiter itself fails, allow the request but log.
      // Consumer should provide logging in production; keep this minimal here.
      // eslint-disable-next-line no-console
      console.error('Rate limiter error:', err);
      return next();
    }
  };
}
```

### Redis store adapter (optional)

```ts
// redisStore.ts
import IORedis from 'ioredis';
import { Store, TokenRecord } from './rateLimiter';

export function createRedisStore(redis: IORedis.Redis, windowMs: number): Store {
  // This adapter uses a fixed-window counter and does not implement token-bucket refill.
  // It provides a simple compatibility layer for the middleware by exposing get/set.

  return {
    get: async (key: string) => {
      const data = await redis.get(key);
      if (!data) return null;
      try {
        const parsed = JSON.parse(data) as TokenRecord;
        return parsed;
      } catch (err) {
        return null;
      }
    },
    set: async (key: string, record: TokenRecord) => {
      // store the record and set TTL to cover multiple windows; TTL is windowMs * 2 / 1000
      await redis.set(key, JSON.stringify(record), 'PX', Math.max(1000, windowMs * 2));
    },
  };
}
```

Notes: The Redis adapter provided is intentionally simple (persisting the TokenRecord). For more efficient behavior one could implement an atomic Lua script to perform refill and decrement in Redis to avoid race conditions in high concurrency scenarios. If we adopt Redis extensively, we should replace this naive adapter with a Lua-based token bucket or use a sliding-window approach.

## Example usage (Express)

```ts
import express from 'express';
import { rateLimiter, createInMemoryStore } from './rateLimiter';

const app = express();
const limiter = rateLimiter({
  windowMs: 60_000,
  maxRequests: 100,
  burst: 20,
  keyGenerator: (req) => (req.user && req.user.id) ? String(req.user.id) : (req.ip || 'anon'),
  store: createInMemoryStore(),
  onLimitReached: (req, res) => {
    // notify metrics
  },
});

app.use(limiter);

app.get('/', (req, res) => {
  res.send('ok');
});

app.listen(3000);
```

## Tests (Jest) - Example

```ts
// rateLimiter.test.ts
import request from 'supertest';
import express from 'express';
import { rateLimiter } from './rateLimiter';

describe('rate limiter middleware', () => {
  it('allows requests under the limit', async () => {
    const app = express();
    app.use(rateLimiter({ windowMs: 1000, maxRequests: 5, burst: 0 }));
    app.get('/', (req, res) => res.send('ok'));

    for (let i = 0; i < 5; i++) {
      const res = await request(app).get('/');
      expect(res.status).toBe(200);
    }
  });

  it('blocks requests over the limit', async () => {
    const app = express();
    app.use(rateLimiter({ windowMs: 1000, maxRequests: 2, burst: 0 }));
    app.get('/', (req, res) => res.send('ok'));

    await request(app).get('/');
    await request(app).get('/');
    const res = await request(app).get('/');
    expect(res.status).toBe(429);
    expect(res.body).toHaveProperty('retryAfter');
  });
});
```

## Implementation notes

- The in-memory store is fine for single-node deployments and functional tests. For clustered apps, configure the Redis store.
- Edge cases: system clock jumps (NTP sync) can affect refill logic. If precise token accounting is required, prefer Redis+Lua atomic logic.
- The middleware attempts to be fault-tolerant: on internal errors it allows requests rather than blocking traffic.
- Response headers are basic (`X-RateLimit-Limit`, `X-RateLimit-Remaining`, `Retry-After`). We can extend to the RFC-standard `RateLimit-*` headers if desired.

## Configuration and setup

- Install dependencies for TypeScript implementation:
  - npm i express
  - npm i -D typescript ts-node @types/express
- Optional Redis adapter requires `ioredis`:
  - npm i ioredis
- Run tests with Jest:
  - npm i -D jest ts-jest @types/jest supertest @types/supertest
  - Configure jest to use ts-jest or compile before running tests

## Next steps and considerations

- Replace naive Redis adapter with Lua-based token bucket for atomicity in production high-concurrency scenarios.
- Add metrics integration (Prometheus counter/gauge) and alerts for when many clients hit limits.
- Consider multiple limiter strategies (sliding-window, leaky-bucket) as configurable policies.

---

If you'd like I can also split this into separate files, add a package.json, and provide CI test config. — Francisco
