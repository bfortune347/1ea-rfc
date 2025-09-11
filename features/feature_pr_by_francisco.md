# CSV Export Feature

## Feature overview

This markdown contains a complete implementation for a streaming CSV export endpoint with a per-user in-memory rate limiter. The goal is to provide a safe, memory-efficient way to export large datasets as CSV while protecting the service from abuse.

The implementation is written for a Node.js + Express + TypeScript stack but is easy to adapt to JavaScript or other frameworks.

## Technical specifications

- Endpoint: GET /api/v1/exports/csv
- Query parameters:
  - columns (optional): comma-separated list of columns to include. Default: all available columns.
  - apiKey (required for rate limiting demo): used as the user identifier for rate limiting.
- Response: streaming CSV with Content-Type: text/csv and Content-Disposition: attachment; filename="export.csv"
- Rate limiting: sliding window per API key. Default: 60 requests per 1 minute.
- Backpressure: Respect writable stream drain; pause source while the response buffer is full.

## Files included (in this markdown for demonstration)

- package.json
- tsconfig.json
- src/app.ts
- src/middleware/rateLimiter.ts
- src/services/exportService.ts
- src/routes/export.ts
- tests/exportService.test.ts

## package.json

```json
{
  "name": "csv-export-example",
  "version": "1.0.0",
  "main": "dist/app.js",
  "scripts": {
    "build": "tsc",
    "start": "ts-node src/app.ts",
    "test": "jest"
  },
  "dependencies": {
    "express": "^4.18.2",
    "csv-stringify": "^6.2.0"
  },
  "devDependencies": {
    "@types/express": "^4.17.17",
    "ts-node": "^10.9.1",
    "typescript": "^4.9.5",
    "jest": "^29.5.0",
    "@types/jest": "^29.5.0",
    "ts-jest": "^29.0.5"
  }
}
```

## tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "outDir": "dist",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true
  },
  "include": ["src/**/*"]
}
```

## src/app.ts

```ts
import express from 'express'
import exportRouter from './routes/export'

const app = express()
app.use(express.json())

app.use('/api/v1/exports', exportRouter)

const port = process.env.PORT || 3000
app.listen(Number(port), () => {
  console.log('CSV export service running on port', port)
})

export default app
```

## src/middleware/rateLimiter.ts

```ts
import { Request, Response, NextFunction } from 'express'

type Key = string

interface WindowEntry {
  timestamp: number
}

interface Bucket {
  entries: number[]
}

// Simple in-memory sliding window rate limiter. For production, use Redis or another
// shared store for consistency across multiple instances.
export function createRateLimiter(maxRequests = 60, windowMs = 60_000) {
  const buckets = new Map<Key, Bucket>()

  function cleanup() {
    const now = Date.now()
    for (const [key, bucket] of buckets) {
      // drop empty or old buckets to keep memory small
      if (bucket.entries.length === 0 || now - bucket.entries[bucket.entries.length - 1] > windowMs * 2) {
        buckets.delete(key)
      }
    }
  }

  setInterval(cleanup, windowMs * 2).unref()

  return function rateLimiterMiddleware(req: Request, res: Response, next: NextFunction) {
    const apiKey = (req.query.apiKey as string) || req.header('x-api-key') || 'anonymous'
    const now = Date.now()
    const windowStart = now - windowMs

    const bucket = buckets.get(apiKey) || { entries: [] }
    // drop old timestamps
    bucket.entries = bucket.entries.filter(ts => ts >= windowStart)

    if (bucket.entries.length >= maxRequests) {
      res.status(429).json({ error: 'Too many requests - rate limit exceeded' })
      return
    }

    bucket.entries.push(now)
    buckets.set(apiKey, bucket)
    next()
  }
}
```

## src/services/exportService.ts

```ts
import { Readable } from 'stream'
import stringify from 'csv-stringify'

export type Row = Record<string, string | number | null>

// Simulated data source that yields rows asynchronously.
// Replace this with a DB cursor or a streaming API in production.
export async function* simulateDataSource(total = 10000) {
  for (let i = 0; i < total; i++) {
    // Simulate some latency
    if (i % 1000 === 0) await new Promise(resolve => setTimeout(resolve, 0))
    yield {
      id: i + 1,
      name: `user-${i + 1}`,
      value: (Math.random() * 1000).toFixed(2),
      created_at: new Date().toISOString()
    }
  }
}

// Create a readable stream from an async iterator
export function streamFromAsyncIterator(iter: AsyncIterable<Row>): Readable {
  const iterator = iter[Symbol.asyncIterator]()
  return new Readable({
    objectMode: true,
    async read() {
      try {
        const result = await iterator.next()
        if (result.done) {
          this.push(null)
        } else {
          this.push(result.value)
        }
      } catch (err) {
        this.destroy(err as Error)
      }
    }
  })
}

// Orchestrates streaming rows as CSV to a writable stream (response).
export async function streamCsvToResponse(
  rowsAsyncIter: AsyncIterable<Row>,
  res: any,
  columns?: string[]
): Promise<void> {
  const source = streamFromAsyncIterator(rowsAsyncIter)

  // configure csv-stringify
  const stringifier = stringify({ header: true, columns: columns })

  // Pipe source -> object to stringifier
  const objToCsv = source.pipe(stringifier)

  return new Promise((resolve, reject) => {
    // Pipe to response and honor backpressure. express res is a Writable stream.
    objToCsv.on('error', err => {
      try {
        if (!res.headersSent) {
          res.status(500).send('error,internal_server_error')
        } else {
          // write a short CSV error row to keep parsers deterministic
          try { res.write('\n"__ERROR__","internal_server_error"') } catch (_) {}
        }
      } catch (_) {}
      reject(err)
    })

    res.on('close', () => {
      objToCsv.destroy(new Error('client closed connection'))
      reject(new Error('client closed connection'))
    })

    objToCsv.pipe(res)

    res.on('finish', () => resolve())
  })
}
```

## src/routes/export.ts

```ts
import express from 'express'
import { createRateLimiter } from '../middleware/rateLimiter'
import { simulateDataSource, streamCsvToResponse } from '../services/exportService'

const router = express.Router()

// Use defaults suitable for demo. In production these should be env-driven.
const rateLimiter = createRateLimiter(60, 60_000)

router.get('/csv', rateLimiter, async (req, res) => {
  try {
    const columnsParam = (req.query.columns as string) || ''
    const columns = columnsParam ? columnsParam.split(',').map(c => c.trim()) : undefined

    res.setHeader('Content-Type', 'text/csv')
    res.setHeader('Content-Disposition', 'attachment; filename="export.csv"')

    // In a real app, replace simulateDataSource with a DB cursor/async iterator.
    const total = Number(req.query.total) || 10000
    const dataIter = simulateDataSource(total)

    await streamCsvToResponse(dataIter, res, columns)
  } catch (err) {
    console.error('Export failed:', err)
    if (!res.headersSent) res.status(500).json({ error: 'internal_server_error' })
  }
})

export default router
```

## tests/exportService.test.ts

```ts
import { streamFromAsyncIterator, streamCsvToResponse } from '../src/services/exportService'
import { PassThrough } from 'stream'

describe('exportService streaming', () => {
  test('streamCsvToResponse completes and writes CSV header', async () => {
    // Create a small async iterator
    async function* smallIter() {
      yield { id: 1, name: 'alice', value: 12 }
      yield { id: 2, name: 'bob', value: 34 }
    }

    const res = new PassThrough()
    const chunks: Buffer[] = []
    res.on('data', c => chunks.push(Buffer.from(c)))

    // Simulate express res by setting headersSent to false and providing write/end
    ;(res as any).headersSent = false

    await streamCsvToResponse(smallIter(), res, ['id', 'name', 'value'])

    const body = Buffer.concat(chunks).toString('utf8')
    expect(body).toContain('id,name,value')
    expect(body).toContain('alice')
    expect(body).toContain('bob')
  })
})
```

## Implementation notes

- The rate limiter is intentionally simple and in-memory for clarity. To make it robust under multiple API server instances, replace the bucket storage with Redis and use atomic operations for sliding window management.
- The data source is simulated by simulateDataSource. For large production exports from Postgres, use pg-query-stream with a client pool and proper row parsing.
- csv-stringify is used to handle escaping and consistent CSV formatting. The columns parameter can be passed to control output order.
- Backpressure: by connecting the Node readable stream through csv-stringify to the response, Node stream backpressure is respected. The implementation also listens for response close to abort work early.

## Configuration / setup requirements

1. Node.js >= 14
2. Install dependencies: npm install
3. Run in development: npm run start
4. Run tests: npm run test

## Example requests

- Download default export of 10000 rows:

```bash
curl -v "http://localhost:3000/api/v1/exports/csv?apiKey=test-key" -o export.csv
```

- Request a filtered columns set:

```bash
curl "http://localhost:3000/api/v1/exports/csv?apiKey=test-key&columns=id,name"
```

- Request a smaller export for local testing:

```bash
curl "http://localhost:3000/api/v1/exports/csv?apiKey=test-key&total=100"
```

## Next steps / production hardening

- Replace in-memory rate limiter with Redis-based implementation for horizontal scaling.
- Add authentication middleware for stronger API key validation and quota enforcement.
- Monitor exporter performance and memory usage under real load and tune connection and stream timeouts.
- Add streaming compression (gzip) if clients accept it; be careful with backpressure implications.


Thank you,

Francisco

