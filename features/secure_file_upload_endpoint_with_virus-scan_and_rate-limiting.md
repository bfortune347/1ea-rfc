# Secure File Upload Feature

This document describes the secure file upload feature implemented by Francisco.

Feature overview
- Endpoint: POST /api/uploads
- Purpose: Accept user file uploads, scan them for viruses, and store only files that pass scan
- Protections: rate limiting, mime type and size validation, virus scanning, temporary-upload staging

Technical specifications
- Language: Node.js (CommonJS)
- Web framework: express
- Multipart parser: multer
- Virus scanning: clamscan CLI invoked via child_process
- Rate limiting: express-rate-limit
- File limits: max 10 MB per file
- Allowed mime types: image/png, image/jpeg, application/pdf

Directory layout (excerpt)
- server.js            # express app and routes
- scanner.js           # wrapper around clamscan CLI
- uploads/             # permanent storage for clean files
- tmp_uploads/         # temporary staging directory
- tests/               # unit and integration tests
- docker-compose.yml   # optional ClamAV service for local testing

Usage and configuration
- Environment variables (with defaults):
  - UPLOAD_TMP_DIR: './tmp_uploads'
  - UPLOAD_DIR: './uploads'
  - MAX_FILE_SIZE_BYTES: 10485760   # 10 MB
  - RATE_LIMIT_WINDOW_MS: 60000
  - RATE_LIMIT_MAX: 10
  - CLAMSCAN_PATH: 'clamscan'       # path to clamscan binary or a wrapper

Implementation notes
- Files are accepted into a tmp directory. After successful scan they are moved to final storage.
- The scanner module returns either 'clean' or 'infected' with details. On infected files, the temporary file is deleted.
- The endpoint responds with 201 and JSON meta on success, 400 for validation errors, 422 for infected files, 429 for rate limit, 500 for server errors.

Code

server.js

```js
const express = require('express')
const path = require('path')
const fs = require('fs')
const util = require('util')
const multer = require('multer')
const rateLimit = require('express-rate-limit')
const scanner = require('./scanner')

const rename = util.promisify(fs.rename)
const unlink = util.promisify(fs.unlink)
const mkdir = util.promisify(fs.mkdir)

const UPLOAD_TMP_DIR = process.env.UPLOAD_TMP_DIR || './tmp_uploads'
const UPLOAD_DIR = process.env.UPLOAD_DIR || './uploads'
const MAX_FILE_SIZE_BYTES = parseInt(process.env.MAX_FILE_SIZE_BYTES || '10485760', 10)

async function ensureDirs () {
  for (const dir of [UPLOAD_TMP_DIR, UPLOAD_DIR]) {
    try {
      await mkdir(dir, { recursive: true })
    } catch (err) {
      // ignore
    }
  }
}

const app = express()

const uploadLimiter = rateLimit({
  windowMs: parseInt(process.env.RATE_LIMIT_WINDOW_MS || '60000', 10),
  max: parseInt(process.env.RATE_LIMIT_MAX || '10', 10),
  standardHeaders: true,
  legacyHeaders: false
})

const storage = multer.diskStorage({
  destination: function (req, file, cb) {
    cb(null, UPLOAD_TMP_DIR)
  },
  filename: function (req, file, cb) {
    const unique = Date.now() + '-' + Math.round(Math.random() * 1e9)
    cb(null, unique + '-' + file.originalname)
  }
})

const allowedMimes = new Set(['image/png', 'image/jpeg', 'application/pdf'])

function fileFilter (req, file, cb) {
  if (!allowedMimes.has(file.mimetype)) {
    return cb(new Error('invalid mime type'))
  }
  cb(null, true)
}

const upload = multer({ storage, fileFilter, limits: { fileSize: MAX_FILE_SIZE_BYTES } })

app.post('/api/uploads', uploadLimiter, upload.single('file'), async (req, res) => {
  if (!req.file) {
    return res.status(400).json({ error: 'no file provided' })
  }

  const tmpPath = req.file.path
  try {
    const result = await scanner.scanFile(tmpPath)
    if (result.status === 'infected') {
      await unlink(tmpPath)
      return res.status(422).json({ error: 'file infected', details: result.details })
    }

    const destPath = path.join(UPLOAD_DIR, path.basename(tmpPath))
    await rename(tmpPath, destPath)
    return res.status(201).json({ message: 'file uploaded', filename: path.basename(destPath) })
  } catch (err) {
    try { await unlink(tmpPath) } catch (e) { }
    return res.status(500).json({ error: 'scan failed', details: err.message })
  }
})

// health endpoint
app.get('/health', (req, res) => {
  res.status(200).json({ status: 'ok' })
})

async function start (port) {
  await ensureDirs()
  const p = port || process.env.PORT || 3000
  return new Promise((resolve) => {
    const s = app.listen(p, () => resolve(s))
  })
}

if (require.main === module) {
  start().then(() => console.log('server started'))
}

module.exports = { app, start }
```

scanner.js

```js
const { execFile } = require('child_process')
const util = require('util')
const execFileAsync = util.promisify(execFile)

const CLAMSCAN_PATH = process.env.CLAMSCAN_PATH || 'clamscan'

// scanFile returns a promise resolving to { status: 'clean' } or { status: 'infected', details }
async function scanFile (filePath) {
  // clamscan returns exit code 0 for no virus, 1 for virus, >1 for errors
  // sample command: clamscan --fdpass --no-summary /path/to/file
  try {
    const args = ['--no-summary', filePath]
    const { stdout, stderr } = await execFileAsync(CLAMSCAN_PATH, args, { timeout: 120000 })
    // stdout example: '/tmp/foo: OK\n'
    const out = String(stdout || '')
    if (/OK$/.test(out.trim())) {
      return { status: 'clean' }
    }
    // some clamscan versions include 'FOUND' text
    if (/FOUND/.test(out)) {
      return { status: 'infected', details: out.trim() }
    }
    // fallback: if exit code was 0 but no OK, still treat as clean
    return { status: 'clean' }
  } catch (err) {
    // err may have code 1 for infected with stdout available
    if (err && err.stdout && /FOUND/.test(String(err.stdout))) {
      return { status: 'infected', details: String(err.stdout).trim() }
    }
    throw new Error('clam scan failed: ' + (err && err.message ? err.message : 'unknown'))
  }
}

module.exports = { scanFile }
```

tests/scanner.test.js

```js
const fs = require('fs')
const path = require('path')
const { execFile } = require('child_process')
const scanner = require('../scanner')

// These are unit tests which mock execFile to avoid calling real clamscan

jest.mock('child_process', () => ({
  execFile: jest.fn()
}))

describe('scanner', () => {
  it('returns clean when clamscan output OK', async () => {
    execFile.mockImplementation((cmd, args, opts, cb) => {
      cb(null, '/tmp/file: OK\n', '')
    })
    const res = await scanner.scanFile('/tmp/file')
    expect(res.status).toBe('clean')
  })

  it('returns infected when clamscan outputs FOUND', async () => {
    execFile.mockImplementation((cmd, args, opts, cb) => {
      const error = new Error('exit code 1')
      error.stdout = '/tmp/file: EICAR-Test-Signature FOUND\n'
      cb(error, error.stdout, '')
    })
    const res = await scanner.scanFile('/tmp/file')
    expect(res.status).toBe('infected')
  })
})
```

tests/integration.test.js

```js
const request = require('supertest')
const fs = require('fs')
const path = require('path')
const { app, start } = require('../server')

let server
beforeAll(async () => {
  server = await start(0)
})

afterAll(async () => {
  await new Promise((resolve) => server.close(resolve))
})

test('upload flow returns 201 on clean file', async () => {
  // For integration tests that run real clamscan, ensure clamscan is installed or mock scanner
  const smallFile = Buffer.from('hello world')
  const res = await request(app)
    .post('/api/uploads')
    .attach('file', smallFile, 'hello.txt')
    .expect(201)
  expect(res.body.filename).toBeDefined()
})
```

Docker compose for local testing of clamd

```yaml
version: '3.7'
services:
  clamd:
    image: mkodockx/docker-clamav:alpine
    restart: unless-stopped
    ports:
      - '3310:3310'
    volumes:
      - clamdb:/var/lib/clamav
volumes:
  clamdb:
```

Notes about running locally
- Option A: Install clamscan on host and ensure linux user has access. Set CLAMSCAN_PATH if not on PATH.
- Option B: Use docker-compose above to run a clamd service and adjust scanner.js to use a network client such as clamdjs; current implementation uses clamscan CLI so provide a local clamscan binary.

Manual quickstart
1) npm install dependencies: express multer express-rate-limit supertest jest
2) create directories: tmp_uploads and uploads or let server create them
3) ensure clamscan is available in PATH or set CLAMSCAN_PATH env var
4) start server: node server.js
5) POST form-data to /api/uploads with key 'file'

Implementation considerations and next steps
- Replace CLI scanner with a TCP client to clamd for better performance and to integrate with the dockerized clamd service
- Add virus quarantine strategy instead of immediate deletion for infected files if traceability is required
- Add monitoring and metrics for scan durations, failures, and rate-limit hits

Contact
- Implementation and design by Francisco
