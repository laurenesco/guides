
## Table of Contents
 
1. [Mental Model — The Biggest Conceptual Shift](#1-mental-model--the-biggest-conceptual-shift)
2. [Architecture](#2-architecture)
3. [Async & I/O](#3-async--io)
4. [Express.js Routing & Middleware](#4-expressjs-routing--middleware)
5. [Security Fundamentals](#5-security-fundamentals)
6. [Financial Application Rules](#6-financial-application-rules)
7. [Recommended Stack & Tooling](#7-recommended-stack--tooling)
---
 
## 1. Mental Model — The Biggest Conceptual Shift
 
| | LAMP + Perl CGI | Node.js |
|---|---|---|
| **Concurrency model** | Apache spawns a new process per request | One long-lived process handles all requests |
| **Execution model** | Script runs top-to-bottom, then exits | Single JS thread with an event loop |
| **Memory** | Resets between requests | Persists — and can leak |
| **Blocking I/O** | Fine — the process is disposable | Freezes ALL requests — never block |
| **Concurrency** | More processes/threads | Async callbacks/promises on one thread |
| **Worker threads** | N/A | Exist but rarely needed |
 
> ⚠️ **The event loop is sacred.** If your code blocks for 200ms (CPU-heavy work, synchronous file reads, tight loops), every other request waits. This is the #1 gotcha coming from CGI.
 
### How the Event Loop Works (Simplified)
 
Node has one JS thread. I/O (DB queries, HTTP calls, file reads) is handed off to the OS asynchronously. When the OS is done, a callback is queued. The event loop picks up callbacks between requests. This is why `async/await` exists — it lets you write sequential-looking code that actually yields control while waiting.
 
### CommonJS vs ES Modules
 
You'll see two module systems. CommonJS (`require()`) is the legacy default. ES Modules (`import/export`) is the modern standard. For new projects, use ES Modules — add `"type": "module"` to `package.json`.
 
```js
// CommonJS (older)
const express = require('express');
module.exports = { handler };
 
// ES Modules (modern — prefer this)
import express from 'express';
export { handler };
```
 
---
 
## 2. Architecture
 
### Recommended Folder Structure
 
```
my-fintech-api/
├── src/
│   ├── routes/          # Express routers — one file per resource
│   │   ├── accounts.js
│   │   └── transactions.js
│   ├── controllers/     # Request/response handling logic
│   ├── services/        # Business logic (pure functions, no req/res)
│   ├── models/          # DB schema + queries (or ORM models)
│   ├── middleware/      # Auth, validation, error handling
│   ├── config/          # DB config, env vars, constants
│   └── app.js           # Express app setup (no listen() here)
├── server.js            # Entry point — calls app.listen()
├── .env                 # Secrets — NEVER commit this
└── package.json
```
 
> Separating `app.js` from `server.js` makes testing easier — you can import the app without starting the server.
 
### The Request Lifecycle
 
```
Incoming HTTP request
  → Express router
  → middleware chain
  → controller
  → service
  → model / DB
  → response
```
 
Each layer has a single responsibility.
 
```js
// routes/accounts.js
import { Router } from 'express';
import { getAccount } from '../controllers/accounts.js';
import { requireAuth } from '../middleware/auth.js';
 
const router = Router();
router.get('/:id', requireAuth, getAccount);
export default router;
 
// controllers/accounts.js
export async function getAccount(req, res, next) {
  try {
    const account = await AccountService.findById(req.params.id);
    res.json(account);
  } catch (err) { next(err); }
}
```
 
### Environment Variables
 
Use the `dotenv` package. Never hardcode secrets. Always validate env vars at startup so the server fails fast rather than failing in production mid-request.
 
```js
// config/env.js — validate at startup
import 'dotenv/config';
const required = ['DATABASE_URL', 'JWT_SECRET', 'ENCRYPTION_KEY'];
for (const key of required) {
  if (!process.env[key]) throw new Error(`Missing env var: ${key}`);
}
```
 
---
 
## 3. Async & I/O
 
### Evolution: Callbacks → Promises → async/await
 
Always use `async/await` in new code. It's syntactic sugar over Promises and is the most readable pattern.
 
```js
// Old callback style (avoid)
db.query(sql, (err, rows) => {
  if (err) return handleError(err);
  doSomething(rows);
});
 
// Modern async/await (use this)
async function getTransactions(accountId) {
  const rows = await db.query(
    'SELECT * FROM transactions WHERE account_id = $1',
    [accountId]
  );
  return rows;
}
```
 
### Parallel vs Sequential Async
 
Don't await one thing, then await another if they're independent. Run them in parallel with `Promise.all`.
 
```js
// Slow — sequential (200ms + 150ms = 350ms)
const account = await getAccount(id);
const profile = await getProfile(id);
 
// Fast — parallel (max(200ms, 150ms) = 200ms)
const [account, profile] = await Promise.all([
  getAccount(id),
  getProfile(id),
]);
```
 
### Never Block the Event Loop
 
> 🚫 These block ALL requests — avoid in request handlers: `fs.readFileSync`, `crypto.pbkdf2Sync`, JSON parsing of huge payloads, tight CPU loops, `child_process.execSync`.
 
```js
// Bad — blocks event loop
const hash = crypto.pbkdf2Sync(password, salt, 100000, 64, 'sha512');
 
// Good — yields control while hashing
const hash = await bcrypt.hash(password, 12); // bcrypt is async-native
 
// For CPU-heavy work — offload to a worker thread
import { Worker } from 'worker_threads';
```
 
### Always Handle Promise Rejections
 
```js
// Catch in async route handlers
app.get('/account/:id', async (req, res, next) => {
  try {
    const data = await getAccount(req.params.id);
    res.json(data);
  } catch (err) {
    next(err); // passes to Express error handler
  }
});
 
// Global safety net (should not be your only handler)
process.on('unhandledRejection', (reason) => {
  console.error('Unhandled rejection:', reason);
});
```
 
---
 
## 4. Express.js Routing & Middleware
 
### Middleware is Your CGI Pipeline Equivalent
 
In CGI, you'd run checks at the top of each script. In Express, middleware functions are chained. Each calls `next()` to pass control, or sends a response to short-circuit.
 
```js
// Middleware signature: (req, res, next) => void
export function requireAuth(req, res, next) {
  const token = req.headers.authorization?.split(' ')[1];
  if (!token) return res.status(401).json({ error: 'Unauthorized' });
  try {
    req.user = jwt.verify(token, process.env.JWT_SECRET);
    next();
  } catch {
    res.status(401).json({ error: 'Invalid token' });
  }
}
 
// Apply globally or per-route
app.use('/api/accounts', requireAuth, accountsRouter);
```
 
### Input Validation with Zod
 
Never trust request data. Validate and type-check every incoming body, query, and param. Zod is the current best-in-class library for this.
 
```js
import { z } from 'zod';
 
const TransferSchema = z.object({
  fromAccountId: z.string().uuid(),
  toAccountId: z.string().uuid(),
  amount: z.number().positive().multipleOf(0.01),
  currency: z.enum(['USD', 'EUR', 'GBP']),
});
 
export async function transfer(req, res, next) {
  const result = TransferSchema.safeParse(req.body);
  if (!result.success)
    return res.status(400).json({ errors: result.error.flatten() });
  // result.data is now fully typed and validated
}
```
 
### Error Handling Middleware
 
One global error handler at the bottom of your middleware stack. Four parameters signals to Express this is an error handler.
 
```js
// src/middleware/errorHandler.js
export function errorHandler(err, req, res, next) {
  const status = err.status || 500;
  console.error(err); // log full error server-side
  res.status(status).json({
    error: status < 500 ? err.message : 'Internal server error'
    // Never leak stack traces to clients in production
  });
}
 
// app.js — must be LAST
app.use(errorHandler);
```
 
---
 
## 5. Security Fundamentals
 
| Area | Tool / Practice |
|---|---|
| **Auth tokens** | Short-lived JWTs (15 min) + httpOnly cookie refresh tokens |
| **Security headers** | `helmet` — one line sets 15+ headers |
| **Rate limiting** | `express-rate-limit` — tightest on auth endpoints |
| **CORS** | `cors` package with explicit allowlist — never `origin: '*'` |
| **SQL injection** | Parameterized queries or ORM always — never concatenate user input |
| **Dependency audits** | `npm audit` regularly; `npm ci` in CI/CD |
 
### Baseline Security Setup
 
```js
import helmet from 'helmet';
import cors from 'cors';
import rateLimit from 'express-rate-limit';
import express from 'express';
 
const app = express();
 
app.use(helmet());
app.use(cors({ origin: process.env.ALLOWED_ORIGINS?.split(',') }));
app.use(express.json({ limit: '10kb' })); // limit body size
 
const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 10,
  message: { error: 'Too many attempts, please try later' }
});
app.use('/api/auth', authLimiter);
```
 
> 🚫 **Never log sensitive data:** passwords, full card numbers, SSNs, API keys, JWT tokens. Log request IDs for correlation, not PII. This is both a security requirement and often a regulatory one (PCI-DSS, SOC 2).
 
---
 
## 6. Financial Application Rules
 
### Never Use Floating Point for Money
 
> 🚫 `0.1 + 0.2 === 0.30000000000000004` in JavaScript. This is catastrophic for financial calculations.
 
Store all monetary values as integers (cents/minor currency units) in the database. Use a library like `dinero.js` or `decimal.js` for arithmetic.
 
```js
// Bad — floating point error
const total = 1.10 + 2.20; // 3.3000000000000003
 
// Good — integer cents
const totalCents = 110 + 220; // 330 cents = $3.30
const display = (totalCents / 100).toFixed(2); // "3.30"
 
// Better — use Dinero.js
import { dinero, add, toDecimal } from 'dinero.js';
import { USD } from '@dinero.js/currencies';
const a = dinero({ amount: 110, currency: USD });
const b = dinero({ amount: 220, currency: USD });
const total = add(a, b); // Dinero({ amount: 330 })
```
 
### Idempotency for Transactions
 
Network failures cause retries. A payment endpoint hit twice must not charge twice. Use idempotency keys — the client generates a unique key per request, you store it in the DB and return the same result on duplicates.
 
```js
// Client sends: X-Idempotency-Key: uuid-v4-per-operation
export async function processPayment(req, res, next) {
  const key = req.headers['x-idempotency-key'];
  if (!key) return res.status(400).json({ error: 'Idempotency key required' });
 
  const existing = await db.query(
    'SELECT response FROM idempotency_keys WHERE key = $1', [key]
  );
  if (existing.rows[0]) return res.json(existing.rows[0].response);
 
  // Process and store result atomically in a DB transaction
}
```
 
### Database Transactions for Financial Operations
 
Any operation that modifies multiple rows must run in a database transaction. A transfer that debits one account and credits another must be atomic.
 
```js
export async function transfer({ fromId, toId, amountCents }) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    await client.query(
      'UPDATE accounts SET balance = balance - $1 WHERE id = $2 AND balance >= $1',
      [amountCents, fromId]
    );
    await client.query(
      'UPDATE accounts SET balance = balance + $1 WHERE id = $2',
      [amountCents, toId]
    );
    await client.query('COMMIT');
  } catch (err) {
    await client.query('ROLLBACK');
    throw err;
  } finally {
    client.release();
  }
}
```
 
### Audit Logging
 
Every state-changing operation on financial data must be logged: who did it, what changed, when, and from what IP. This is non-negotiable for compliance and forensics.
 
```js
// audit_log table: user_id, action, entity_type, entity_id,
//   old_value (JSONB), new_value (JSONB), ip_address, timestamp
await db.query(
  `INSERT INTO audit_log (user_id, action, entity_type, entity_id, old_value, new_value, ip_address)
   VALUES ($1,$2,$3,$4,$5,$6,$7)`,
  [req.user.id, 'TRANSFER', 'account', fromId, oldBalance, newBalance, req.ip]
);
```
 
---
 
## 7. Recommended Stack & Tooling
 
| Category | Recommended | Alternatives |
|---|---|---|
| **Framework** | Express.js | Fastify, Hono |
| **Database** | PostgreSQL + `pg` / `postgres.js` | Prisma ORM, Drizzle ORM |
| **Validation** | Zod | Joi |
| **Auth** | `jsonwebtoken`, `bcrypt` / `argon2` | Passport.js |
| **Logging** | `pino` (structured JSON) | Winston |
| **Testing** | Vitest + Supertest | Jest |
| **Money** | `dinero.js` | `decimal.js` |
| **Security** | `helmet`, `express-rate-limit`, `cors` | — |
| **Process mgmt** | PM2 (production), nodemon (dev) | — |
 
### Essential `package.json` Scripts
 
```json
{
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js",
    "test": "vitest",
    "test:coverage": "vitest --coverage",
    "lint": "eslint src/",
    "audit": "npm audit --audit-level=high"
  }
}
```
 
### Structured Logging with Pino
 
In production, logs should be machine-readable JSON, not pretty console strings. This feeds into log aggregation systems (Datadog, CloudWatch, etc.).
 
```js
import pino from 'pino';
export const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  redact: ['req.headers.authorization', '*.password', '*.ssn']
});
 
// Use structured fields, not string interpolation
logger.info({ userId: req.user.id, action: 'transfer', amount: 100 }, 'Transfer initiated');
// Not: logger.info(`User ${userId} transferred $100`);
```
 
> ✅ **Quick start:** Run `npm init -y`, then install `express`, `zod`, `helmet`, `cors`, `express-rate-limit`, `pino`, `dotenv`, and `pg`. That's enough to build a production-quality financial API skeleton.
 
---
 
*Guide covers Node.js fundamentals for experienced web developers migrating from LAMP/Perl CGI to a Node.js financial backend.*
 

