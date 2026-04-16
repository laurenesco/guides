# Node.js Testing Guide

> Focused on Vitest + Supertest for a financial Express/Node.js backend. Covers unit tests, integration tests, mocking, test databases, and best practices.

---

## Table of Contents

1. [Testing Philosophy](#1-testing-philosophy)
2. [Setup & Configuration](#2-setup--configuration)
3. [Folder Structure](#3-folder-structure)
4. [Unit Tests](#4-unit-tests)
5. [Integration Tests](#5-integration-tests)
6. [Mocking](#6-mocking)
7. [Test Database Strategy](#7-test-database-strategy)
8. [Testing Auth & Middleware](#8-testing-auth--middleware)
9. [Financial-Specific Testing](#9-financial-specific-testing)
10. [Coverage](#10-coverage)
11. [Running Tests in CI/CD](#11-running-tests-in-cicd)
12. [Quick Reference](#12-quick-reference)

---

## 1. Testing Philosophy

### The Testing Pyramid

```
        /\
       /  \
      / E2E \          <- Few. Slow. Test full user flows.
     /--------\
    /Integration\      <- Some. Test HTTP + real DB together.
   /--------------\
  /   Unit Tests   \   <- Many. Fast. Test logic in isolation.
 /------------------\
```

For a financial backend, aim for this rough split:

| Layer | What it tests | Speed | Quantity |
|---|---|---|---|
| **Unit** | Pure functions, services, validators | Milliseconds | Most of your tests |
| **Integration** | HTTP routes, middleware, real DB | Seconds | Key flows and edge cases |
| **E2E** | Full user journeys across services | Slow | Critical paths only |

### Core principles

**Test behavior, not implementation.** Tests should describe what the code *does*, not how it does it internally. If you refactor a function without changing its behavior, no tests should break.

**Test the unhappy paths as hard as the happy paths.** For financial code, the edge cases — insufficient funds, duplicate transactions, expired tokens, malformed amounts — are where real damage happens.

**Never test against your development database.** Tests must be isolated, repeatable, and safe to run destructively.

**Aim for high coverage on your services layer.** That is where business logic lives. Controllers and routes are well-covered by integration tests. Do not chase 100% as a vanity metric.

---

## 2. Setup & Configuration

### Install dependencies

```bash
# Core testing tools
npm install --save-dev vitest supertest

# Coverage
npm install --save-dev @vitest/coverage-v8

# Optional: for type checking in tests
npm install --save-dev @types/supertest
```

### `vitest.config.js`

```js
// vitest.config.js
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    setupFiles: ['./tests/setup.js'],   // runs before every test file
    environment: 'node',
    globals: false,                      // prefer explicit imports over globals
    testTimeout: 10000,                  // 10s timeout — integration tests can be slow
    coverage: {
      provider: 'v8',
      reporter: ['text', 'html', 'lcov'],
      exclude: [
        'node_modules/',
        'tests/',
        'src/config/',                   // config files aren't logic to cover
        '**/*.test.js',
      ],
      thresholds: {
        lines: 80,
        functions: 80,
        branches: 75,
      },
    },
  },
});
```

### `tests/setup.js`

Runs once before all tests. Sets up env vars so the real `.env` is never loaded during testing.

```js
// tests/setup.js
import { beforeAll, afterAll } from 'vitest';
import { pool } from '../src/config/db.js';

beforeAll(() => {
  // Override env vars for test environment
  process.env.NODE_ENV = 'test';
  process.env.DATABASE_URL = process.env.TEST_DATABASE_URL;
  process.env.JWT_SECRET = 'test-secret-do-not-use-in-production';
  process.env.ENCRYPTION_KEY = 'test-encryption-key-32-chars-min';
  process.env.LOG_LEVEL = 'silent';    // suppress logs during tests
});

afterAll(async () => {
  await pool.end();                    // close DB pool cleanly after all tests
});
```

### `package.json` scripts

```json
{
  "scripts": {
    "test": "vitest run",
    "test:watch": "vitest",
    "test:coverage": "vitest run --coverage",
    "test:unit": "vitest run tests/unit",
    "test:integration": "vitest run tests/integration"
  }
}
```

---

## 3. Folder Structure

Mirror your `src/` structure inside `tests/` so it is obvious which test covers which source file.

```
tests/
├── setup.js                           # global setup — runs before all tests
├── helpers/
│   ├── auth.js                        # shared helper: generate test JWTs
│   ├── factories.js                   # shared helper: create test DB records
│   └── db.js                          # shared helper: transaction wrapper
├── unit/
│   ├── services/
│   │   ├── transfer.test.js           # mirrors src/services/transfer.js
│   │   └── accounts.test.js
│   ├── middleware/
│   │   └── auth.test.js
│   └── validators/
│       └── transfer.test.js
└── integration/
    ├── accounts.test.js               # mirrors src/routes/accounts.js
    └── transactions.test.js
```

---

## 4. Unit Tests

Unit tests cover a single function or module in complete isolation. No HTTP, no real database, no file system. Dependencies are mocked.

### Basic structure

```js
// tests/unit/services/transfer.test.js
import { describe, it, expect, beforeEach } from 'vitest';
import { validateTransfer } from '../../../src/services/transfer.js';

describe('validateTransfer', () => {

  it('accepts a valid transfer', () => {
    expect(() => validateTransfer({
      fromAccountId: 'uuid-a',
      toAccountId: 'uuid-b',
      amountCents: 1000,
      currency: 'USD',
    })).not.toThrow();
  });

  it('rejects a negative amount', () => {
    expect(() => validateTransfer({
      fromAccountId: 'uuid-a',
      toAccountId: 'uuid-b',
      amountCents: -50,
      currency: 'USD',
    })).toThrow('Amount must be positive');
  });

  it('rejects zero amount', () => {
    expect(() => validateTransfer({
      fromAccountId: 'uuid-a',
      toAccountId: 'uuid-b',
      amountCents: 0,
      currency: 'USD',
    })).toThrow('Amount must be positive');
  });

  it('rejects transfers to the same account', () => {
    expect(() => validateTransfer({
      fromAccountId: 'uuid-a',
      toAccountId: 'uuid-a',
      amountCents: 1000,
      currency: 'USD',
    })).toThrow('Cannot transfer to same account');
  });

  it('rejects unsupported currencies', () => {
    expect(() => validateTransfer({
      fromAccountId: 'uuid-a',
      toAccountId: 'uuid-b',
      amountCents: 1000,
      currency: 'XYZ',
    })).toThrow('Unsupported currency');
  });

});
```

### Testing async service functions

```js
// tests/unit/services/accounts.test.js
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { getAccountById } from '../../../src/services/accounts.js';
import { pool } from '../../../src/config/db.js';

// Mock the DB pool so no real DB connection is needed
vi.mock('../../../src/config/db.js', () => ({
  pool: { query: vi.fn() }
}));

describe('getAccountById', () => {

  beforeEach(() => {
    vi.clearAllMocks();
  });

  it('returns an account when found', async () => {
    pool.query.mockResolvedValue({
      rows: [{ id: 'uuid-a', balance: 50000, currency: 'USD' }]
    });

    const account = await getAccountById('uuid-a');
    expect(account.id).toBe('uuid-a');
    expect(account.balance).toBe(50000);
  });

  it('throws NotFoundError when account does not exist', async () => {
    pool.query.mockResolvedValue({ rows: [] });

    await expect(getAccountById('uuid-missing'))
      .rejects.toThrow('Account not found');
  });

  it('queries by the correct account ID', async () => {
    pool.query.mockResolvedValue({ rows: [{ id: 'uuid-a' }] });

    await getAccountById('uuid-a');

    expect(pool.query).toHaveBeenCalledWith(
      expect.stringContaining('WHERE id = $1'),
      ['uuid-a']
    );
  });

});
```

---

## 5. Integration Tests

Integration tests fire real HTTP requests against your Express app using Supertest, against a real test database. They test the full stack — routing, middleware, controllers, services, and DB — working together.

### The pattern

```js
// tests/integration/accounts.test.js
import { describe, it, expect, beforeAll, afterAll, beforeEach } from 'vitest';
import request from 'supertest';
import app from '../../src/app.js';
import { pool } from '../../src/config/db.js';
import { generateTestToken } from '../helpers/auth.js';
import { createTestAccount } from '../helpers/factories.js';

let client;

// Before this file's tests: get a dedicated client and begin a transaction
beforeAll(async () => {
  client = await pool.connect();
  await client.query('BEGIN');
});

// After this file's tests: roll back everything — DB is clean for the next run
afterAll(async () => {
  await client.query('ROLLBACK');
  client.release();
});

describe('GET /api/accounts/:id', () => {

  it('returns 401 with no token', async () => {
    const res = await request(app).get('/api/accounts/some-id');
    expect(res.status).toBe(401);
  });

  it('returns 401 with an invalid token', async () => {
    const res = await request(app)
      .get('/api/accounts/some-id')
      .set('Authorization', 'Bearer not-a-real-token');
    expect(res.status).toBe(401);
  });

  it('returns 404 for a non-existent account', async () => {
    const token = generateTestToken({ userId: 'user-1' });
    const res = await request(app)
      .get('/api/accounts/00000000-0000-0000-0000-000000000000')
      .set('Authorization', `Bearer ${token}`);
    expect(res.status).toBe(404);
  });

  it('returns the account for an authenticated owner', async () => {
    const account = await createTestAccount(client, {
      userId: 'user-1',
      balanceCents: 50000,
      currency: 'USD',
    });
    const token = generateTestToken({ userId: 'user-1' });

    const res = await request(app)
      .get(`/api/accounts/${account.id}`)
      .set('Authorization', `Bearer ${token}`);

    expect(res.status).toBe(200);
    expect(res.body.id).toBe(account.id);
    expect(res.body.balanceCents).toBe(50000);
  });

  it('returns 403 when a user requests another user\'s account', async () => {
    const account = await createTestAccount(client, { userId: 'user-2' });
    const token = generateTestToken({ userId: 'user-1' }); // different user

    const res = await request(app)
      .get(`/api/accounts/${account.id}`)
      .set('Authorization', `Bearer ${token}`);

    expect(res.status).toBe(403);
  });

});
```

---

## 6. Mocking

### Mock a module with `vi.mock`

```js
import { vi } from 'vitest';

// Mock an entire module — must be at the top level, not inside a test
vi.mock('../../../src/config/db.js', () => ({
  pool: {
    query: vi.fn(),
    connect: vi.fn(),
  }
}));
```

### Spy on a function without replacing it

```js
import { vi, expect } from 'vitest';
import * as transferService from '../../../src/services/transfer.js';

const spy = vi.spyOn(transferService, 'processTransfer');

// Assert it was called
expect(spy).toHaveBeenCalledOnce();
expect(spy).toHaveBeenCalledWith({ fromId: 'a', toId: 'b', amountCents: 1000 });
```

### Mock resolved/rejected values

```js
// Simulate a successful DB response
pool.query.mockResolvedValue({ rows: [{ id: 'uuid-a' }] });

// Simulate a DB error
pool.query.mockRejectedValue(new Error('Connection timeout'));

// Different response on each call
pool.query
  .mockResolvedValueOnce({ rows: [{ balance: 50000 }] })  // first call
  .mockResolvedValueOnce({ rows: [] });                    // second call
```

### Reset mocks between tests

```js
import { beforeEach, vi } from 'vitest';

beforeEach(() => {
  vi.clearAllMocks();     // clears call history and return values
  // vi.resetAllMocks();  // also resets implementation
  // vi.restoreAllMocks(); // restores spies to original implementation
});
```

### Mock environment variables

```js
beforeEach(() => {
  vi.stubEnv('JWT_SECRET', 'test-secret');
  vi.stubEnv('NODE_ENV', 'test');
});

afterEach(() => {
  vi.unstubAllEnvs();
});
```

---

## 7. Test Database Strategy

### Use a separate test database

Your `.env.test` should point at a completely separate database from development:

```bash
# .env.test
DATABASE_URL=postgres://user:password@localhost:5432/myapp_test
NODE_ENV=test
JWT_SECRET=test-secret-not-for-production
LOG_LEVEL=silent
```

### The transaction rollback pattern

Wrap each test file's DB operations in a transaction and roll it back after. The database is always returned to a clean state without needing to re-seed or truncate tables.

```js
// tests/helpers/db.js
import { pool } from '../../src/config/db.js';

export async function withTestTransaction(fn) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    await fn(client);
  } finally {
    await client.query('ROLLBACK');
    client.release();
  }
}
```

### Test data factories

Rather than duplicating `INSERT` statements across tests, use factory helpers that create realistic test records with sensible defaults:

```js
// tests/helpers/factories.js
import { randomUUID } from 'crypto';

export async function createTestUser(client, overrides = {}) {
  const defaults = {
    id: randomUUID(),
    email: `test-${Date.now()}@example.com`,
    passwordHash: 'hashed-password',
    role: 'user',
  };
  const data = { ...defaults, ...overrides };

  const { rows } = await client.query(
    `INSERT INTO users (id, email, password_hash, role)
     VALUES ($1, $2, $3, $4) RETURNING *`,
    [data.id, data.email, data.passwordHash, data.role]
  );
  return rows[0];
}

export async function createTestAccount(client, overrides = {}) {
  const defaults = {
    id: randomUUID(),
    userId: randomUUID(),
    balanceCents: 100000,
    currency: 'USD',
    status: 'active',
  };
  const data = { ...defaults, ...overrides };

  const { rows } = await client.query(
    `INSERT INTO accounts (id, user_id, balance_cents, currency, status)
     VALUES ($1, $2, $3, $4, $5) RETURNING *`,
    [data.id, data.userId, data.balanceCents, data.currency, data.status]
  );
  return rows[0];
}
```

---

## 8. Testing Auth & Middleware

### Generate test tokens

A shared helper that creates valid JWTs for test users, so you do not hardcode tokens across test files:

```js
// tests/helpers/auth.js
import jwt from 'jsonwebtoken';

export function generateTestToken(payload = {}, options = {}) {
  const defaults = {
    userId: 'test-user-id',
    role: 'user',
  };
  return jwt.sign(
    { ...defaults, ...payload },
    process.env.JWT_SECRET,
    { expiresIn: '1h', ...options }
  );
}

export function generateExpiredToken(payload = {}) {
  return generateTestToken(payload, { expiresIn: '-1s' });
}
```

### Testing middleware in isolation

```js
// tests/unit/middleware/auth.test.js
import { describe, it, expect, vi } from 'vitest';
import { requireAuth } from '../../../src/middleware/auth.js';
import { generateTestToken, generateExpiredToken } from '../../helpers/auth.js';

function mockReqRes(authHeader) {
  const req = { headers: { authorization: authHeader } };
  const res = {
    status: vi.fn().mockReturnThis(),
    json: vi.fn().mockReturnThis(),
  };
  const next = vi.fn();
  return { req, res, next };
}

describe('requireAuth middleware', () => {

  it('calls next() with a valid token', () => {
    const token = generateTestToken({ userId: 'user-1' });
    const { req, res, next } = mockReqRes(`Bearer ${token}`);

    requireAuth(req, res, next);

    expect(next).toHaveBeenCalledOnce();
    expect(req.user.userId).toBe('user-1');
  });

  it('returns 401 with no Authorization header', () => {
    const { req, res, next } = mockReqRes(undefined);

    requireAuth(req, res, next);

    expect(res.status).toHaveBeenCalledWith(401);
    expect(next).not.toHaveBeenCalled();
  });

  it('returns 401 with an expired token', () => {
    const token = generateExpiredToken({ userId: 'user-1' });
    const { req, res, next } = mockReqRes(`Bearer ${token}`);

    requireAuth(req, res, next);

    expect(res.status).toHaveBeenCalledWith(401);
  });

  it('returns 401 with a malformed token', () => {
    const { req, res, next } = mockReqRes('Bearer not.a.token');

    requireAuth(req, res, next);

    expect(res.status).toHaveBeenCalledWith(401);
  });

});
```

---

## 9. Financial-Specific Testing

### Test money arithmetic carefully

```js
// tests/unit/services/transfer.test.js
describe('transfer amount calculations', () => {

  it('does not use floating point for currency', async () => {
    // $1.10 + $2.20 should be exactly $3.30, not $3.3000000000000003
    const result = await calculateTotal([110, 220]); // amounts in cents
    expect(result).toBe(330);
    expect(result).not.toBeCloseTo(330.0000001);
  });

  it('handles large amounts without precision loss', async () => {
    const result = await calculateTotal([999999999, 1]); // ~$10M in cents
    expect(result).toBe(1000000000);
  });

});
```

### Test idempotency

```js
describe('POST /api/transfers — idempotency', () => {

  it('requires an idempotency key', async () => {
    const token = generateTestToken();
    const res = await request(app)
      .post('/api/transfers')
      .set('Authorization', `Bearer ${token}`)
      .send({ fromAccountId: 'a', toAccountId: 'b', amountCents: 1000 });

    expect(res.status).toBe(400);
    expect(res.body.error).toMatch(/idempotency/i);
  });

  it('returns the same result when called twice with the same key', async () => {
    const token = generateTestToken();
    const idempotencyKey = randomUUID();
    const payload = {
      fromAccountId: testAccountA.id,
      toAccountId: testAccountB.id,
      amountCents: 1000,
    };

    const first = await request(app)
      .post('/api/transfers')
      .set('Authorization', `Bearer ${token}`)
      .set('X-Idempotency-Key', idempotencyKey)
      .send(payload);

    const second = await request(app)
      .post('/api/transfers')
      .set('Authorization', `Bearer ${token}`)
      .set('X-Idempotency-Key', idempotencyKey)
      .send(payload);

    expect(first.status).toBe(200);
    expect(second.status).toBe(200);
    expect(second.body.transferId).toBe(first.body.transferId); // same result
  });

});
```

### Test DB transaction atomicity

```js
describe('transfer — atomicity', () => {

  it('rolls back both accounts if the credit fails', async () => {
    const initialBalance = testAccountA.balanceCents;

    // Force the second query (credit) to fail by mocking after the debit
    vi.spyOn(pool, 'query')
      .mockResolvedValueOnce({ rows: [] })  // debit succeeds
      .mockRejectedValueOnce(new Error('DB error'));  // credit fails

    await expect(processTransfer({
      fromId: testAccountA.id,
      toId: testAccountB.id,
      amountCents: 1000,
    })).rejects.toThrow();

    // Balance should be unchanged — rollback worked
    const { rows } = await pool.query(
      'SELECT balance_cents FROM accounts WHERE id = $1',
      [testAccountA.id]
    );
    expect(rows[0].balance_cents).toBe(initialBalance);
  });

  it('rejects a transfer when balance is insufficient', async () => {
    const account = await createTestAccount(client, { balanceCents: 500 });

    await expect(processTransfer({
      fromId: account.id,
      toId: otherAccount.id,
      amountCents: 1000, // more than available
    })).rejects.toThrow('Insufficient funds');
  });

});
```

### Test input validation thoroughly

```js
describe('POST /api/transfers — validation', () => {

  const validPayload = {
    fromAccountId: 'uuid-a',
    toAccountId: 'uuid-b',
    amountCents: 1000,
    currency: 'USD',
  };

  const cases = [
    ['missing fromAccountId', { ...validPayload, fromAccountId: undefined }],
    ['missing toAccountId', { ...validPayload, toAccountId: undefined }],
    ['non-integer amount', { ...validPayload, amountCents: 10.50 }],
    ['negative amount', { ...validPayload, amountCents: -100 }],
    ['zero amount', { ...validPayload, amountCents: 0 }],
    ['unsupported currency', { ...validPayload, currency: 'XYZ' }],
    ['non-UUID account ID', { ...validPayload, fromAccountId: 'not-a-uuid' }],
  ];

  it.each(cases)('returns 400 for %s', async (label, payload) => {
    const token = generateTestToken();
    const res = await request(app)
      .post('/api/transfers')
      .set('Authorization', `Bearer ${token}`)
      .send(payload);

    expect(res.status).toBe(400);
  });

});
```

---

## 10. Coverage

Run coverage reports to find untested paths:

```bash
npm run test:coverage
```

This generates a report in `coverage/` — open `coverage/index.html` in a browser for a line-by-line breakdown.

### Interpreting coverage

| Metric | What it measures |
|---|---|
| **Lines** | Were these lines executed at all? |
| **Functions** | Was each function called at least once? |
| **Branches** | Were both sides of each `if/else` tested? |
| **Statements** | Similar to lines — each statement executed? |

Branch coverage is the most valuable for financial code. An `if (balance >= amount)` with only the `true` branch tested means the insufficient-funds path is untested.

### What to exclude from coverage

```js
// vitest.config.js
coverage: {
  exclude: [
    'src/config/',         // env, db setup — not business logic
    'src/server.js',       // entry point
    '**/*.test.js',
    '**/index.js',         // barrel re-export files
  ]
}
```

---

## 11. Running Tests in CI/CD

On Render (or any CI platform), run tests before deploy. A failed test should block the deploy.

### Example GitHub Actions workflow

```yaml
# .github/workflows/test.yml
name: Test

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
          POSTGRES_DB: myapp_test
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    env:
      NODE_ENV: test
      TEST_DATABASE_URL: postgres://testuser:testpass@localhost:5432/myapp_test
      JWT_SECRET: ci-test-secret
      ENCRYPTION_KEY: ci-test-encryption-key-32-chars

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci
      - run: npm test
      - run: npm run test:coverage
```

---

## 12. Quick Reference

### Vitest API cheat sheet

```js
// Structure
describe('group name', () => { ... });
it('test name', () => { ... });
it.each(cases)('test %s', (label, input) => { ... });
it.skip('skipped test', ...);
it.only('run only this test', ...);

// Lifecycle hooks
beforeAll(() => { ... });   // once before all tests in the file
afterAll(() => { ... });    // once after all tests in the file
beforeEach(() => { ... });  // before each test
afterEach(() => { ... });   // after each test

// Assertions
expect(value).toBe(exact);
expect(value).toEqual(deepEqual);
expect(value).toBeNull();
expect(value).toBeTruthy();
expect(value).toBeFalsy();
expect(value).toContain(item);
expect(value).toHaveLength(n);
expect(value).toMatchObject({ partial: 'match' });
expect(fn).toThrow('message');
await expect(promise).rejects.toThrow('message');
await expect(promise).resolves.toBe(value);

// Mocks
vi.fn()                           // create a mock function
vi.mock('module-path', factory)   // mock an entire module
vi.spyOn(object, 'method')        // spy on a method
vi.clearAllMocks()                // clear call history
vi.resetAllMocks()                // clear + reset implementation
vi.restoreAllMocks()              // restore spies to originals
mock.mockResolvedValue(val)       // async mock return value
mock.mockRejectedValue(err)       // async mock error
mock.mockResolvedValueOnce(val)   // single-use async return
expect(mock).toHaveBeenCalled()
expect(mock).toHaveBeenCalledWith(args)
expect(mock).toHaveBeenCalledOnce()
```

### Common Supertest patterns

```js
// GET
const res = await request(app).get('/api/accounts/123');

// POST with body
const res = await request(app)
  .post('/api/transfers')
  .send({ fromAccountId: 'a', toAccountId: 'b', amountCents: 1000 });

// With auth header
const res = await request(app)
  .get('/api/accounts/123')
  .set('Authorization', `Bearer ${token}`);

// With custom headers
const res = await request(app)
  .post('/api/transfers')
  .set('Authorization', `Bearer ${token}`)
  .set('X-Idempotency-Key', uuid)
  .send(payload);

// Assert response
expect(res.status).toBe(200);
expect(res.body).toMatchObject({ id: 'uuid-a' });
expect(res.headers['content-type']).toMatch(/json/);
```
