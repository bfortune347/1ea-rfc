# Bulk User Import Feature

## Feature overview

This document and code provide a complete implementation of a new API endpoint to import users in bulk into our PostgreSQL database while preserving idempotency, validation, and transactional safety.

Key points:
- Endpoint: POST /api/users/bulk
- Idempotency via X-Request-ID header
- Server-side validation with Joi
- Transactional upsert to avoid duplicates (unique on email)
- Configurable batch size

## Technical specifications

- Node.js (>=14, recommended 18+)
- Express
- pg (node-postgres)
- Joi for validation
- Jest + Supertest for tests

Environment variables:
- DATABASE_URL: Postgres connection string (required)
- PORT: server port (default 3000)
- IMPORT_BATCH_SIZE: number of rows per batch insert (default 500)

## Database migrations

File: migrations/20250910_create_users_and_import_jobs.sql

```sql
-- Create users table
CREATE TABLE IF NOT EXISTS users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  email TEXT NOT NULL UNIQUE,
  metadata JSONB,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT now(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT now()
);

-- Create import_jobs table for idempotency tracking
CREATE TABLE IF NOT EXISTS import_jobs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  request_id UUID NOT NULL UNIQUE,
  status TEXT NOT NULL CHECK (status IN ('pending', 'completed', 'failed')),
  summary JSONB,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT now(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT now()
);

-- Index for quick lookup
CREATE INDEX IF NOT EXISTS idx_import_jobs_request_id ON import_jobs(request_id);
```

Note: gen_random_uuid() requires the pgcrypto extension. If using uuid-ossp, replace accordingly.

## Implementation

Below are the main code files. Adjust paths as needed.

Project structure (relevant files):

- package.json
- src/server.js
- src/db/index.js
- src/routes/users.js
- src/services/bulkImportService.js
- src/models/userRepository.js
- migrations/20250910_create_users_and_import_jobs.sql
- tests/bulk_import.test.js
- docker-compose.yml

### package.json

```json
{
  "name": "bulk-import-service",
  "version": "1.0.0",
  "main": "src/server.js",
  "scripts": {
    "start": "node src/server.js",
    "test": "jest --runInBand"
  },
  "dependencies": {
    "express": "^4.18.2",
    "joi": "^17.9.1",
    "pg": "^8.11.0",
    "uuid": "^9.0.0"
  },
  "devDependencies": {
    "jest": "^29.0.0",
    "supertest": "^6.3.3"
  }
}
```

### src/db/index.js

```js
const { Pool } = require('pg');

const pool = new Pool({
  connectionString: process.env.DATABASE_URL
});

module.exports = {
  query: (text, params) => pool.query(text, params),
  getClient: () => pool.connect()
};
```

### src/routes/users.js

```js
const express = require('express');
const router = express.Router();
const Joi = require('joi');
const bulkImportService = require('../services/bulkImportService');

const userSchema = Joi.object({
  name: Joi.string().min(1).max(256).required(),
  email: Joi.string().email().required(),
  metadata: Joi.object().optional()
});

const payloadSchema = Joi.array().items(userSchema).min(1).max(10000).required();

router.post('/bulk', async (req, res) => {
  const requestIdHeader = req.header('X-Request-ID');
  const requestId = requestIdHeader || null;

  const { error, value } = payloadSchema.validate(req.body);
  if (error) {
    return res.status(400).json({ error: error.message });
  }

  try {
    const result = await bulkImportService.importUsers({
      requestId,
      users: value
    });

    return res.status(200).json(result);
  } catch (err) {
    console.error('Bulk import failed', err);
    return res.status(500).json({ error: 'Internal Server Error' });
  }
});

module.exports = router;
```

### src/models/userRepository.js

```js
const db = require('../db');

async function insertUsersTransaction(client, users) {
  if (!users || users.length === 0) return { inserted: 0, updated: 0 };

  // Build multi-row insert with ON CONFLICT (email) DO UPDATE
  const values = [];
  const placeholders = users.map((u, i) => {
    const idx = i * 3; // three columns per user: name,email,metadata
    values.push(u.name, u.email, u.metadata ? JSON.stringify(u.metadata) : null);
    return `($${idx + 1}, $${idx + 2}, $${idx + 3})`;
  }).join(', ');

  const sql = `
    INSERT INTO users (name, email, metadata, created_at, updated_at)
    VALUES ${placeholders}
    ON CONFLICT (email) DO UPDATE
      SET name = EXCLUDED.name,
          metadata = EXCLUDED.metadata,
          updated_at = now()
    RETURNING (xmax = 0) AS inserted_flag;
  `;

  const res = await client.query(sql, values);

  let inserted = 0;
  let updated = 0;
  for (const row of res.rows) {
    if (row.inserted_flag) inserted++; else updated++;
  }

  return { inserted, updated };
}

module.exports = { insertUsersTransaction };
```

Notes: The RETURNING expression (xmax = 0) is a common trick to detect inserts vs updates; depending on Postgres version it should work. If not desired, you can separately query counts or use other techniques.

### src/services/bulkImportService.js

```js
const db = require('../db');
const { insertUsersTransaction } = require('../models/userRepository');
const { v4: uuidv4 } = require('uuid');

const DEFAULT_BATCH = parseInt(process.env.IMPORT_BATCH_SIZE || '500', 10);

async function importUsers({ requestId, users }) {
  // If no request id provided, generate one but mark it as non-repeatable in docs.
  const rid = requestId || uuidv4();

  const client = await db.getClient();
  try {
    await client.query('BEGIN');

    // Check if a job with this request id already exists
    const existing = await client.query('SELECT id, status, summary FROM import_jobs WHERE request_id = $1', [rid]);
    if (existing.rows.length > 0) {
      const job = existing.rows[0];
      if (job.status === 'completed') {
        await client.query('COMMIT');
        return { requestId: rid, id: job.id, status: job.status, summary: job.summary, duplicate: true };
      }
      // If pending or failed, we proceed (we might allow re-run)
    } else {
      // Create a pending job record
      await client.query(
        'INSERT INTO import_jobs (request_id, status, summary) VALUES ($1, $2, $3)',
        [rid, 'pending', JSON.stringify({ total: users.length })]
      );
    }

    // Process in batches
    let totalInserted = 0;
    let totalUpdated = 0;

    for (let i = 0; i < users.length; i += DEFAULT_BATCH) {
      const batch = users.slice(i, i + DEFAULT_BATCH);
      const result = await insertUsersTransaction(client, batch);
      totalInserted += result.inserted;
      totalUpdated += result.updated;
    }

    const summary = { total: users.length, inserted: totalInserted, updated: totalUpdated };

    // Update the job record to completed
    await client.query('UPDATE import_jobs SET status = $1, summary = $2, updated_at = now() WHERE request_id = $3', [
      'completed', JSON.stringify(summary), rid
    ]);

    await client.query('COMMIT');

    // Return a useful summary
    const jobRow = await db.query('SELECT id FROM import_jobs WHERE request_id = $1', [rid]);
    const jobId = jobRow.rows[0].id;

    return { requestId: rid, id: jobId, status: 'completed', summary };
  } catch (err) {
    await client.query('ROLLBACK');

    // Set job as failed (best-effort)
    try {
      await db.query('UPDATE import_jobs SET status = $1, summary = $2, updated_at = now() WHERE request_id = $3', [
        'failed', JSON.stringify({ error: err.message }), rid
      ]);
    } catch (updateErr) {
      console.error('Failed to mark job failed', updateErr);
    }

    throw err;
  } finally {
    client.release();
  }
}

module.exports = { importUsers };
```

### src/server.js

```js
const express = require('express');
const bodyParser = require('body-parser');
const usersRouter = require('./routes/users');

const app = express();
app.use(bodyParser.json({ limit: '10mb' }));

app.use('/api/users', usersRouter);

const port = process.env.PORT || 3000;
app.listen(port, () => {
  console.log(`Bulk import service listening on port ${port}`);
});

module.exports = app;
```

## Tests

### tests/bulk_import.test.js

```js
const request = require('supertest');
const app = require('../src/server');
const db = require('../src/db');

beforeAll(async () => {
  // Ensure tables exist for integration tests
  await db.query(`
    CREATE TABLE IF NOT EXISTS users (
      id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
      name TEXT NOT NULL,
      email TEXT NOT NULL UNIQUE,
      metadata JSONB,
      created_at TIMESTAMP WITH TIME ZONE DEFAULT now(),
      updated_at TIMESTAMP WITH TIME ZONE DEFAULT now()
    );
    CREATE TABLE IF NOT EXISTS import_jobs (
      id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
      request_id UUID NOT NULL UNIQUE,
      status TEXT NOT NULL CHECK (status IN ('pending', 'completed', 'failed')),
      summary JSONB,
      created_at TIMESTAMP WITH TIME ZONE DEFAULT now(),
      updated_at TIMESTAMP WITH TIME ZONE DEFAULT now()
    );
  `);
});

afterAll(async () => {
  // Cleanup
  await db.query('DROP TABLE IF EXISTS import_jobs');
  await db.query('DROP TABLE IF EXISTS users');
  await db.query('END');
});

describe('POST /api/users/bulk', () => {
  test('imports users and is idempotent with same request id', async () => {
    const payload = [
      { name: 'Alice', email: 'alice@example.com' },
      { name: 'Bob', email: 'bob@example.com' }
    ];

    const requestId = '11111111-1111-1111-1111-111111111111';

    const res1 = await request(app)
      .post('/api/users/bulk')
      .set('X-Request-ID', requestId)
      .send(payload)
      .expect(200);

    expect(res1.body).toHaveProperty('status', 'completed');
    expect(res1.body.summary.inserted + res1.body.summary.updated).toBe(2);

    const res2 = await request(app)
      .post('/api/users/bulk')
      .set('X-Request-ID', requestId)
      .send(payload)
      .expect(200);

    expect(res2.body).toHaveProperty('duplicate', true);
    expect(res2.body.summary.inserted + res2.body.summary.updated).toBe(2);
  });

  test('rejects invalid payload', async () => {
    const res = await request(app)
      .post('/api/users/bulk')
      .send([{ name: '', email: 'not-an-email' }])
      .expect(400);

    expect(res.body).toHaveProperty('error');
  });
});
```

Notes: In CI, you should run migrations before tests and use a fresh DB. For unit tests you can mock db client methods.

## docker-compose.yml (for integration testing)

```yaml
version: '3.8'
services:
  db:
    image: postgres:15
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: bulk_import_test
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      retries: 5
  app:
    build: .
    environment:
      DATABASE_URL: postgres://postgres:postgres@db:5432/bulk_import_test
    depends_on:
      db:
        condition: service_healthy
    ports:
      - "3000:3000"
```

## Implementation notes and considerations

- Idempotency: We store request_id in import_jobs with unique constraint. Clients should provide a UUID in X-Request-ID. If omitted, server generates one; however clients should provide it to benefit from deduplication.
- Concurrency: Two simultaneous requests with the same request_id could race. The unique constraint on import_jobs.request_id and our transaction handling should ensure only one job completes normally. Consider explicit SELECT FOR UPDATE on import_jobs when supporting stronger concurrency cases.
- Error handling: The service attempts to mark a job as failed on any error. Some failure paths (e.g., if connection dies) might leave a pending job; consider background reconciliation.
- Large payloads: For very large payloads we should prefer streaming, chunked processing, or a background worker. This implementation supports chunking via IMPORT_BATCH_SIZE.
- Upsert semantics: The ON CONFLICT (email) DO UPDATE helps avoid duplicates by email. If business rules differ (e.g., use idempotent matching on external_id), adjust accordingly.

## Example curl usage

Replace DATABASE_URL and run server first.

```bash
curl -X POST http://localhost:3000/api/users/bulk \
  -H "Content-Type: application/json" \
  -H "X-Request-ID: 123e4567-e89b-12d3-a456-426614174000" \
  -d '[{"name":"Alice","email":"alice@example.com"},{"name":"Bob","email":"bob@example.com"}]'
```

## Setup and run

1. Ensure Postgres is available and DATABASE_URL is set.
2. Apply the migration file in migrations/20250910_create_users_and_import_jobs.sql.
3. npm install
4. npm start
5. Run tests with npm test (ensure test DB configured and migrations applied)

## Final notes

This implementation is intentionally synchronous to provide a clear, atomic import operation with idempotency guarantees. If our expected imports grow (time or size), the next iteration should move processing to an async worker (e.g., background job queue) and provide polling endpoints for job status. If you'd like, I can follow up with a variant that enqueues to Bull/RabbitMQ and implements a status endpoint.

— Francisco
