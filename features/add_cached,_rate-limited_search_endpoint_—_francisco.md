# Feature: Cached, rate-limited search endpoint

## Feature overview
Adds a new `/api/search` endpoint that:
- Applies a per-client rate limit (requests per minute)
- Caches search results in Redis (or in-memory fallback)
- Returns consistent responses and reduces upstream API usage

## Technical specifications
- Endpoint: GET /api/search?q=<query>
- Rate limiting: default 60 requests per minute per IP (configurable via RATE_LIMIT_PER_MINUTE)
- Caching: default TTL 120 seconds (CACHE_TTL_SECONDS)
- Redis used when REDIS_URL is configured, otherwise in-memory fallback is used
- Node 14+, Express 4+, ioredis client

## Configuration / Setup
.env.example:
```env
PORT=3000
RATE_LIMIT_PER_MINUTE=60
CACHE_TTL_SECONDS=120
REDIS_URL=redis://localhost:6379
```

package.json (partial):
```json
{
  "name": "search-service",
  "version": "1.0.0",
  "main": "src/server.js",
  "scripts": {
    "start": "node src/server.js",
    "test": "jest --runInBand"
  },
  "dependencies": {
    "express": "^4.18.2",
    "ioredis": "^5.3.1",
    "node-fetch": "^2.6.7"
  },
  "devDependencies": {
    "jest": "^29.0.0",
    "supertest": "^6.3.1"
  }
}
```

## Implementation notes

src/server.js
```js
const express = require("express");
const Cache = require("./cache");
const rateLimiter = require("./rateLimiter");
const searchHandler = require("./searchHandler");

const app = express();
const cache = new Cache();

app.get("/api/search", rateLimiter.middleware, async (req, res, next) => {
  try {
    const q = (req.query.q || "").trim();
    if (!q) {
      return res.status(400).json({ error: "Missing query parameter 'q'" });
    }
    const cacheKey = `search:${q.toLowerCase()}`;
    const cached = await cache.get(cacheKey);
    if (cached) {
      return res.json({ source: "cache", data: cached });
    }
    const result = await searchHandler.search(q);
    await cache.set(cacheKey, result);
    res.json({ source: "upstream", data: result });
  } catch (err) {
    next(err);
  }
});

if (require.main === module) {
  const port = process.env.PORT || 3000;
  app.listen(port, () => {
    console.log(`Search service listening on ${port}`);
  });
}

module.exports = app;
```

src/cache.js
```js
const Redis = require("ioredis");

class Cache {
  constructor() {
    const redisUrl = process.env.REDIS_URL;
    if (redisUrl) {
      this.client = new Redis(redisUrl);
      this.isRedis = true;
    } else {
      this.store = new Map();
      this.isRedis = false;
    }
    this.ttl = parseInt(process.env.CACHE_TTL_SECONDS || "120", 10);
  }

  async get(key) {
    if (this.isRedis) {
      const raw = await this.client.get(key);
      return raw ? JSON.parse(raw) : null;
    } else {
      const entry = this.store.get(key);
      if (!entry) return null;
      if (Date.now() > entry.expiresAt) {
        this.store.delete(key);
        return null;
      }
      return entry.value;
    }
  }

  async set(key, value) {
    if (this.isRedis) {
      await this.client.set(key, JSON.stringify(value), "EX", this.ttl);
    } else {
      this.store.set(key, {
        value,
        expiresAt: Date.now() + this.ttl * 1000
      });
    }
  }
}

module.exports = Cache;
```

src/rateLimiter.js
```js
const Redis = require("ioredis");

const WINDOW_SECONDS = 60;
const DEFAULT_LIMIT = parseInt(process.env.RATE_LIMIT_PER_MINUTE || "60", 10);

let redisClient;
if (process.env.REDIS_URL) {
  redisClient = new Redis(process.env.REDIS_URL);
}

const inMemory = new Map();

async function incrementRedis(key) {
  const count = await redisClient.incr(key);
  if (count === 1) {
    await redisClient.expire(key, WINDOW_SECONDS);
  }
  return count;
}

function middleware(req, res, next) {
  const ip = req.ip || req.connection.remoteAddress || "unknown";
  const key = `rl:${ip}`;
  if (redisClient) {
    incrementRedis(key)
      .then(count => {
        if (count > DEFAULT_LIMIT) {
          res.set("Retry-After", String(WINDOW_SECONDS));
          return res.status(429).json({ error: "Rate limit exceeded" });
        }
        next();
      })
      .catch(err => {
        next(err);
      });
  } else {
    const now = Math.floor(Date.now() / 1000);
    const entry = inMemory.get(key) || { count: 0, start: now };
    if (now - entry.start >= WINDOW_SECONDS) {
      entry.count = 1;
      entry.start = now;
    } else {
      entry.count++;
    }
    inMemory.set(key, entry);
    if (entry.count > DEFAULT_LIMIT) {
      res.set("Retry-After", String(WINDOW_SECONDS - (now - entry.start)));
      return res.status(429).json({ error: "Rate limit exceeded" });
    }
    next();
  }
}

module.exports = { middleware };
```

src/searchHandler.js
```js
const fetch = require("node-fetch");

async function search(q) {
  // Placeholder: in a real system, call a search API or DB.
  // This function simulates latency and returns deterministic data.
  await new Promise(resolve => setTimeout(resolve, 50));
  return {
    query: q,
    results: [
      { id: 1, title: `Result for ${q} - A` },
      { id: 2, title: `Result for ${q} - B` }
    ],
    fetchedAt: new Date().toISOString()
  };
}

module.exports = { search };
```

tests/search.test.js
```js
const request = require("supertest");
const app = require("../src/server");

describe("GET /api/search", () => {
  it("returns 400 when no q provided", async () => {
    const res = await request(app).get("/api/search");
    expect(res.status).toBe(400);
    expect(res.body.error).toBeTruthy();
  });

  it("returns upstream result and then cache on subsequent call", async () => {
    const res1 = await request(app).get("/api/search").query({ q: "node" });
    expect(res1.status).toBe(200);
    expect(res1.body.source).toBe("upstream");
    const res2 = await request(app).get("/api/search").query({ q: "node" });
    expect(res2.status).toBe(200);
    expect(res2.body.source).toBe("cache");
  });

  it("enforces rate limit (in-memory fallback)", async () => {
    const originalLimit = process.env.RATE_LIMIT_PER_MINUTE;
    process.env.RATE_LIMIT_PER_MINUTE = "3";
    const agent = request.agent(app);
    for (let i = 0; i < 3; i++) {
      const r = await agent.get("/api/search").query({ q: "rl" });
      expect(r.status).toBe(200);
    }
    const blocked = await agent.get("/api/search").query({ q: "rl" });
    expect(blocked.status).toBe(429);
    process.env.RATE_LIMIT_PER_MINUTE = originalLimit;
  });
});
```

README snippet:
```md
# Search Service

Start:
1. Install dependencies: npm install
2. Configure environment (copy .env.example to .env)
3. Start: npm start
4. Run tests: npm test

Notes:
- If REDIS_URL is set, Redis will be used for both caching and rate limiting.
- Defaults provide safe local development behavior with in-memory fallbacks.
```

Implementation notes:
- The rate limiter uses a simple Redis INCR+EXPIRE window. For production, consider a sliding window or token bucket algorithm to provide smoother rate limiting across bursts.
- The cache layer serializes JSON. Be careful with large payloads and memory usage if using the in-memory fallback.
- The search handler is a placeholder; replace with actual upstream API or database calls, add robust error handling and observability (metrics, tracing) as needed.

How I approached this (Francisco):
- Focused on a pragmatic, production-usable default: Redis-backed behavior with clear local development fallbacks.
- Kept the implementation small and testable so it can be iterated on quickly and replaced with a more advanced rate-limiter or cache eviction strategy when load characteristics are better understood.
