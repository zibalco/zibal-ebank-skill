# Next.js App Router

Tested with Next.js 16.4.4, React 19.3.1, better-sqlite3 13.0.3 and zod 4.6.7, on Node.js 22, using `next build` and `next start`. The built-in `fetch` is the HTTP client.

SQLite keeps the example self-contained. It does not suit serverless hosting. Use a real database there and keep the unique indexes and the conditional updates.

Routes are the same as in [node-express.md](node-express.md). They sit at the app root, for example `app/payouts/route.ts`. Moving them under `app/api/` changes only the URLs.

Every route except the webhook needs the `X-Internal-Key` header. It is a placeholder for your own authentication and authorization. Replace it before production. Never call these routes from client components with a key.

Call `POST /internal/reconcile` from a scheduler. The webhook is sent once, so it cannot be the only signal.

## Files

### `package.json`

```json
{
  "name": "zibal-ebank-next",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start"
  },
  "dependencies": {
    "better-sqlite3": "13.0.3",
    "next": "16.4.4",
    "react": "19.3.1",
    "react-dom": "19.3.1",
    "server-only": "0.0.1",
    "zod": "4.6.7"
  },
  "devDependencies": {
    "@types/better-sqlite3": "7.6.13",
    "@types/node": "22.15.3",
    "@types/react": "19.1.2",
    "typescript": "5.8.3"
  }
}
```

### `tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["dom", "dom.iterable", "esnext"],
    "allowJs": false,
    "skipLibCheck": true,
    "strict": true,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "react-jsx",
    "incremental": true,
    "plugins": [{ "name": "next" }],
    "paths": { "@/*": ["./*"] }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts", ".next/dev/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

### `next.config.mjs`

```js
const nextConfig = {
  poweredByHeader: false,
  serverExternalPackages: ['better-sqlite3'],
};

export default nextConfig;
```

### `.env.example`

```bash
# Never prefix any of these with NEXT_PUBLIC_.
ZIBAL_ACCESS_TOKEN=
ZIBAL_ACCOUNT_ID=
# ZIBAL_BASE_URL=https://api.zibal.ir
# ZIBAL_CALLBACK_URL=https://your-domain.example/webhooks/zibal/checkout
# ZIBAL_TIMEOUT_MS=15000
# ZIBAL_REVIEW_AFTER_MS=900000
# ZIBAL_RETURN_WATCH_DAYS=7
INTERNAL_API_KEY=
DB_PATH=./ebank.sqlite
```

### `lib/config.ts`

```ts
import 'server-only';

function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing environment variable ${name}`);
  return value;
}

// Read on first use, because `next build` evaluates route modules without these variables.
export const config = {
  get token() { return required('ZIBAL_ACCESS_TOKEN'); },
  get accountId() { return required('ZIBAL_ACCOUNT_ID'); },
  get internalKey() { return required('INTERNAL_API_KEY'); },
  get baseUrl() { return process.env.ZIBAL_BASE_URL || 'https://api.zibal.ir'; },
  get callbackUrl() { return process.env.ZIBAL_CALLBACK_URL || undefined; },
  get timeoutMs() { return Number(process.env.ZIBAL_TIMEOUT_MS || 15000); },
  get reviewAfterMs() { return Number(process.env.ZIBAL_REVIEW_AFTER_MS || 900000); },
  get returnWatchMs() { return Number(process.env.ZIBAL_RETURN_WATCH_DAYS || 7) * 86400000; },
  get dbPath() { return process.env.DB_PATH || './ebank.sqlite'; },
};
```

### `lib/db.ts`

```ts
import 'server-only';
import Database from 'better-sqlite3';
import { config } from './config';

let connection: Database.Database | undefined;

export function db(): Database.Database {
  if (connection) return connection;
  connection = new Database(config.dbPath);
  connection.pragma('busy_timeout = 5000');
  connection.pragma('journal_mode = WAL');
  connection.exec(`
    CREATE TABLE IF NOT EXISTS payouts (
      id TEXT PRIMARY KEY,
      request_key TEXT NOT NULL UNIQUE,
      request_hash TEXT NOT NULL,
      unique_code TEXT NOT NULL UNIQUE,
      account_id TEXT NOT NULL,
      amount INTEGER NOT NULL,
      iban TEXT NOT NULL,
      reason_code INTEGER NOT NULL,
      delay INTEGER,
      description TEXT,
      payment_number TEXT,
      split INTEGER,
      state TEXT NOT NULL,
      tracker_id TEXT,
      checkouts TEXT NOT NULL DEFAULT '[]',
      last_result INTEGER,
      last_message TEXT,
      error_group TEXT,
      cancel_state TEXT,
      submitted_at INTEGER,
      created_at INTEGER NOT NULL,
      updated_at INTEGER NOT NULL
    );

    CREATE TABLE IF NOT EXISTS deposit_decisions (
      id TEXT PRIMARY KEY,
      account_id TEXT NOT NULL,
      transaction_id TEXT NOT NULL,
      status INTEGER NOT NULL,
      state TEXT NOT NULL,
      refund_tracker_id TEXT,
      last_result INTEGER,
      last_message TEXT,
      submitted_at INTEGER,
      created_at INTEGER NOT NULL,
      updated_at INTEGER NOT NULL,
      UNIQUE (account_id, transaction_id)
    );
  `);
  return connection;
}
```

### `lib/zibal.ts`

```ts
import 'server-only';
import { config } from './config';

export type Reply = { http: number | null; json: any };

// Grouping of the documented result table. See references/result-codes.md.
const RESULT_GROUPS = new Map<number, string>([
  [2, 'auth'], [3, 'auth'], [4, 'auth'], [7, 'auth'], [8, 'auth'],
  [5, 'validation'], [6, 'validation'], [9, 'validation'], [21, 'validation'],
  [44, 'validation'], [48, 'validation'],
  [29, 'funds'],
  [14, 'not_found'],
  [45, 'provider'],
]);
const REJECTION_GROUPS = new Set(['auth', 'validation', 'funds']);
const CHECKOUT_STATUSES = new Set([-1, 0, 3, 4, 5, 20]);
export const PENDING_STATUSES = new Set([-1, 0, 20]);

export function resultGroup(result: number | null): string {
  return (result !== null && RESULT_GROUPS.get(result)) || 'unknown';
}

export function resultOf(reply: Reply): number | null {
  return Number.isInteger(reply.json?.result) ? reply.json.result : null;
}

// True only when Zibal clearly refused the request. Anything else is an unknown outcome.
export function isRejected(reply: Reply): boolean {
  if (reply.http === null || reply.http < 200 || reply.http >= 500) return false;
  return REJECTION_GROUPS.has(resultGroup(resultOf(reply)));
}

async function call(
  method: 'GET' | 'POST',
  path: string,
  { query, body }: { query?: Record<string, string>; body?: unknown } = {},
): Promise<Reply> {
  const url = new URL(path, config.baseUrl);
  for (const [key, value] of Object.entries(query ?? {})) url.searchParams.set(key, value);
  try {
    const response = await fetch(url, {
      method,
      cache: 'no-store',
      redirect: 'manual',
      signal: AbortSignal.timeout(config.timeoutMs),
      headers: {
        Authorization: `Bearer ${config.token}`,
        ...(body ? { 'Content-Type': 'application/json' } : {}),
      },
      body: body ? JSON.stringify(body) : undefined,
    });
    const text = await response.text();
    let json: any = null;
    try {
      json = JSON.parse(text);
    } catch {
      json = null;
    }
    return { http: response.status, json };
  } catch {
    return { http: null, json: null };
  }
}

export const zibal = {
  createCheckout: (body: Record<string, unknown>) => call('POST', '/ebank/v1/account/checkout/create/', { body }),
  inquireCheckout: (uniqueCode: string) => call('GET', '/ebank/v1/account/checkout/inquire/', { query: { uniqueCode } }),
  cancelCheckout: (accountId: string, checkoutIds: string[]) =>
    call('POST', '/ebank/v1/account/checkout/cancel/', { body: { accountId, checkoutIds } }),
  balance: (accountId: string) => call('GET', '/ebank/v1/account/balance/', { query: { accountId } }),
  inquireDeposit: (accountId: string, transactionId: string) =>
    call('GET', '/ebank/v1/account/identified-payment/inquire/', { query: { accountId, transactionId } }),
  changeDepositStatus: (accountId: string, transactionId: string, status: number) =>
    call('POST', '/ebank/v1/account/identified-payment/change-status/', { body: { accountId, transactionId, status } }),
};

// Maps a transfer object to a local state. Every checkout counts, because split creates several.
export function checkoutOutcome(payout: { unique_code: string; amount: number }, data: any): string {
  if (data === null || typeof data !== 'object') return 'needs_review';
  if (data.uniqueCode !== payout.unique_code || data.totalAmount !== payout.amount) return 'needs_review';
  const checkouts = data.checkouts;
  if (!Array.isArray(checkouts) || checkouts.length === 0) return 'needs_review';
  const statuses: unknown[] = checkouts.map((checkout: any) => checkout?.status);
  if (statuses.some((status) => !CHECKOUT_STATUSES.has(status as number))) return 'needs_review';
  if (statuses.some((status) => PENDING_STATUSES.has(status as number))) return 'processing';
  if (statuses.every((status) => status === 3)) {
    return checkouts.every((checkout: any) => checkout.refCode) ? 'succeeded' : 'needs_review';
  }
  if (statuses.every((status) => status === 5)) return 'failed';
  if (statuses.every((status) => status === 4)) return 'returned';
  return 'needs_review';
}

export function maskIban(iban: string): string {
  return `${iban.slice(0, 4)}${'*'.repeat(18)}${iban.slice(-4)}`;
}
```

### `lib/payouts.ts`

```ts
import 'server-only';
import { createHash, randomUUID } from 'node:crypto';
import { config } from './config';
import { db } from './db';
import { checkoutOutcome, isRejected, maskIban, PENDING_STATUSES, resultGroup, resultOf, zibal } from './zibal';

export type Payout = {
  id: string; request_key: string; request_hash: string; unique_code: string; account_id: string;
  amount: number; iban: string; reason_code: number; delay: number | null; description: string | null;
  payment_number: string | null; split: number | null; state: string; tracker_id: string | null;
  checkouts: string; last_result: number | null; error_group: string | null; cancel_state: string | null;
  submitted_at: number | null; created_at: number; updated_at: number;
};

export type PayoutInput = {
  requestKey: string; amount: number; iban: string; reasonCode: number;
  delay?: number; description?: string; paymentNumber?: string; split?: boolean;
};

// A state can only be entered from these states, so a stale read cannot undo a newer one.
const ALLOWED_FROM: Record<string, string[]> = {
  unknown: ['submitting'],
  processing: ['submitting', 'unknown', 'processing', 'needs_review'],
  succeeded: ['submitting', 'unknown', 'processing', 'succeeded', 'needs_review'],
  returned: ['submitting', 'unknown', 'processing', 'succeeded', 'needs_review'],
  failed: ['submitting', 'unknown', 'processing', 'needs_review'],
  needs_review: ['submitting', 'unknown', 'processing', 'succeeded', 'needs_review'],
};
const RECONCILABLE = new Set(['submitting', 'unknown', 'processing', 'succeeded', 'needs_review']);

const log = (event: string, fields: object) => console.info(JSON.stringify({ event, ...fields }));

export class Conflict extends Error {}

export function getPayout(id: string): Payout | undefined {
  return db().prepare('SELECT * FROM payouts WHERE id = ?').get(id) as Payout | undefined;
}

const find = (id: string) => getPayout(id) as Payout;

export function view(payout: Payout, extra: object = {}) {
  return {
    id: payout.id,
    requestKey: payout.request_key,
    uniqueCode: payout.unique_code,
    state: payout.state,
    amount: payout.amount,
    iban: maskIban(payout.iban),
    trackerId: payout.tracker_id,
    lastResult: payout.last_result,
    errorGroup: payout.error_group,
    checkouts: JSON.parse(payout.checkouts),
    ...extra,
  };
}

function transition(id: string, state: string, fields: Record<string, unknown> = {}) {
  const from = ALLOWED_FROM[state];
  const columns: Record<string, unknown> = { ...fields, state, updated_at: Date.now() };
  const assignments = Object.keys(columns).map((column) => `${column} = @${column}`).join(', ');
  const marks = from.map((_, index) => `@from${index}`).join(', ');
  const params: Record<string, unknown> = { ...columns, id };
  from.forEach((value, index) => { params[`from${index}`] = value; });
  db().prepare(`UPDATE payouts SET ${assignments} WHERE id = @id AND state IN (${marks})`).run(params);
}

function checkoutSummary(data: any): string {
  if (!Array.isArray(data?.checkouts)) return '[]';
  return JSON.stringify(data.checkouts.map((checkout: any) => ({
    id: checkout?.id ?? null,
    amount: checkout?.amount ?? null,
    status: checkout?.status ?? null,
    refCode: checkout?.refCode ?? null,
    failReason: checkout?.failReason ?? null,
  })));
}

async function submit(id: string) {
  const now = Date.now();
  const claim = db().prepare(`
    UPDATE payouts SET state = 'submitting', submitted_at = ?, updated_at = ?
    WHERE id = ? AND state = 'created'
  `).run(now, now, id);
  if (claim.changes === 0) return;

  const payout = find(id);
  const callbackUrl = config.callbackUrl;
  const reply = await zibal.createCheckout({
    amount: payout.amount,
    iban: payout.iban,
    accountId: payout.account_id,
    reasonCode: payout.reason_code,
    uniqueCode: payout.unique_code,
    ...(payout.delay !== null ? { delay: payout.delay } : {}),
    ...(payout.description !== null ? { description: payout.description } : {}),
    ...(payout.payment_number !== null ? { paymentNumber: payout.payment_number } : {}),
    ...(payout.split !== null ? { split: payout.split === 1 } : {}),
    ...(callbackUrl ? { callbackUrl } : {}),
  });

  const result = resultOf(reply);
  const base = { last_result: result, last_message: typeof reply.json?.message === 'string' ? reply.json.message : null };

  if (reply.http === 200 && result === 1 && reply.json.data !== null && typeof reply.json.data === 'object') {
    const data = reply.json.data;
    transition(id, checkoutOutcome(payout, data), {
      ...base,
      tracker_id: typeof data.trackerId === 'string' ? data.trackerId : null,
      checkouts: checkoutSummary(data),
    });
  } else if (isRejected(reply)) {
    transition(id, 'failed', { ...base, error_group: resultGroup(result) });
  } else {
    transition(id, 'unknown', { ...base, error_group: resultGroup(result) });
  }
  log('payout_submitted', { id, iban: maskIban(payout.iban), http: reply.http, result, state: find(id).state });
}

export async function createPayout(input: PayoutInput) {
  const hash = createHash('sha256').update(JSON.stringify([
    input.amount, input.iban, input.reasonCode, input.delay ?? null,
    input.description ?? null, input.paymentNumber ?? null, input.split ?? null,
  ])).digest('hex');
  const now = Date.now();
  const id = randomUUID();

  const inserted = db().prepare(`
    INSERT INTO payouts (
      id, request_key, request_hash, unique_code, account_id, amount, iban, reason_code,
      delay, description, payment_number, split, state, created_at, updated_at
    ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, 'created', ?, ?)
    ON CONFLICT (request_key) DO NOTHING
  `).run(
    id, input.requestKey, hash, randomUUID(), config.accountId, input.amount, input.iban, input.reasonCode,
    input.delay ?? null, input.description ?? null, input.paymentNumber ?? null,
    input.split === undefined ? null : Number(input.split), now, now,
  );

  if (inserted.changes === 0) {
    const existing = db().prepare('SELECT * FROM payouts WHERE request_key = ?').get(input.requestKey) as Payout;
    if (existing.request_hash !== hash) throw new Conflict('request_key_reused');
    // 'created' means nothing was sent yet, so the claim in submit() is still safe.
    if (existing.state === 'created') await submit(existing.id);
    return { payout: find(existing.id), duplicate: true };
  }

  await submit(id);
  return { payout: find(id), duplicate: false };
}

export async function reconcilePayout(id: string): Promise<Payout | undefined> {
  const payout = getPayout(id);
  if (!payout || !RECONCILABLE.has(payout.state)) return payout;

  const now = Date.now();
  if (payout.state === 'submitting') {
    // A live request is still inside its timeout. Do not race it.
    if (now - payout.updated_at < config.timeoutMs * 2) return payout;
    transition(id, 'unknown');
  }

  const reply = await zibal.inquireCheckout(payout.unique_code);
  const result = resultOf(reply);
  const base = { last_result: result, last_message: typeof reply.json?.message === 'string' ? reply.json.message : null };

  if (reply.http === 200 && result === 1) {
    const data = reply.json.data;
    transition(id, checkoutOutcome(payout, data), {
      ...base,
      tracker_id: typeof data?.trackerId === 'string' ? data.trackerId : payout.tracker_id,
      checkouts: checkoutSummary(data),
      error_group: null,
    });
  } else {
    const open = payout.state === 'unknown' || payout.state === 'submitting';
    const expired = now - (payout.submitted_at ?? payout.created_at) >= config.reviewAfterMs;
    // Not found for a transfer Zibal already accepted is never normal.
    const vanished = result === 14 && !open && payout.state !== 'needs_review';
    if ((open && expired) || vanished) transition(id, 'needs_review', base);
  }
  log('payout_reconciled', { id, http: reply.http, result, state: find(id).state });
  return find(id);
}

export async function reconcileDue(): Promise<number> {
  const now = Date.now();
  const rows = db().prepare(`
    SELECT id FROM payouts
    WHERE state IN ('unknown', 'processing')
       OR (state = 'submitting' AND updated_at <= ?)
       OR (state = 'succeeded' AND submitted_at >= ?)
    ORDER BY updated_at LIMIT 50
  `).all(now - config.timeoutMs * 2, now - config.returnWatchMs) as { id: string }[];
  for (const row of rows) await reconcilePayout(row.id);
  return rows.length;
}

export async function cancelPayout(payout: Payout) {
  const ids: string[] = JSON.parse(payout.checkouts)
    .filter((checkout: any) => checkout.id && PENDING_STATUSES.has(checkout.status))
    .map((checkout: any) => checkout.id);
  if (ids.length === 0) throw new Conflict('not_cancellable');

  const claim = db().prepare(`
    UPDATE payouts SET cancel_state = 'sent', updated_at = ?
    WHERE id = ? AND cancel_state IS NULL AND state = 'processing'
  `).run(Date.now(), payout.id);
  if (claim.changes === 0) throw new Conflict('not_cancellable');

  const reply = await zibal.cancelCheckout(payout.account_id, ids);
  const result = resultOf(reply);

  if (reply.http === 200 && result === 1) {
    const data = reply.json.data ?? {};
    const updated = (await reconcilePayout(payout.id)) as Payout;
    return {
      http: 200,
      payout: updated,
      cancelOutcome: 'answered',
      cancelled: Array.isArray(data.cancelled_checkouts) ? data.cancelled_checkouts : [],
      uncancellable: Array.isArray(data.uncancellable_checkouts) ? data.uncancellable_checkouts : [],
    };
  }
  if (isRejected(reply)) {
    db().prepare('UPDATE payouts SET cancel_state = NULL, last_result = ?, updated_at = ? WHERE id = ?')
      .run(result, Date.now(), payout.id);
    return { http: 422, payout: find(payout.id), cancelOutcome: 'rejected', cancelled: [], uncancellable: [] };
  }
  // Unknown: keep cancel_state so it cannot be sent twice. Status comes from inquire.
  const updated = (await reconcilePayout(payout.id)) as Payout;
  return { http: 202, payout: updated, cancelOutcome: 'unknown', cancelled: [], uncancellable: [] };
}
```

### `lib/deposits.ts`

```ts
import 'server-only';
import { randomUUID } from 'node:crypto';
import { config } from './config';
import { db } from './db';
import { Conflict } from './payouts';
import { isRejected, resultOf, zibal } from './zibal';

export type Decision = {
  id: string; transaction_id: string; status: number; state: string; refund_tracker_id: string | null;
  last_result: number | null; submitted_at: number | null; created_at: number; updated_at: number;
};

const log = (event: string, fields: object) => console.info(JSON.stringify({ event, ...fields }));

function find(transactionId: string): Decision | undefined {
  return db().prepare('SELECT * FROM deposit_decisions WHERE account_id = ? AND transaction_id = ?')
    .get(config.accountId, transactionId) as Decision | undefined;
}

const must = (transactionId: string) => find(transactionId) as Decision;

export function view(decision: Decision, extra: object = {}) {
  return {
    id: decision.id,
    transactionId: decision.transaction_id,
    status: decision.status,
    state: decision.state,
    refundTrackerId: decision.refund_tracker_id,
    lastResult: decision.last_result,
    ...extra,
  };
}

function setState(id: string, from: string[], state: string, fields: Record<string, unknown> = {}) {
  const columns: Record<string, unknown> = { ...fields, state, updated_at: Date.now() };
  const assignments = Object.keys(columns).map((column) => `${column} = @${column}`).join(', ');
  const marks = from.map((_, index) => `@from${index}`).join(', ');
  const params: Record<string, unknown> = { ...columns, id };
  from.forEach((value, index) => { params[`from${index}`] = value; });
  db().prepare(`UPDATE deposit_decisions SET ${assignments} WHERE id = @id AND state IN (${marks})`).run(params);
}

// Returns the deposit, or null when the answer does not settle anything.
async function fetchDeposit(transactionId: string): Promise<any | null> {
  const reply = await zibal.inquireDeposit(config.accountId, transactionId);
  if (reply.http !== 200 || reply.json?.result !== 1 || !Array.isArray(reply.json.data)) return null;
  return reply.json.data.find((item: any) => item?.id === transactionId) ?? null;
}

export async function decideDeposit(transactionId: string, status: number) {
  const now = Date.now();
  const inserted = db().prepare(`
    INSERT INTO deposit_decisions (id, account_id, transaction_id, status, state, created_at, updated_at)
    VALUES (?, ?, ?, ?, 'created', ?, ?)
    ON CONFLICT (account_id, transaction_id) DO NOTHING
  `).run(randomUUID(), config.accountId, transactionId, status, now, now);

  const decision = must(transactionId);
  if (decision.status !== status) throw new Conflict('different_decision_exists');
  const duplicate = inserted.changes === 0;

  const claim = db().prepare(`
    UPDATE deposit_decisions SET state = 'submitting', updated_at = ?
    WHERE id = ? AND state = 'created'
  `).run(Date.now(), decision.id);
  if (claim.changes === 0) return { http: 200, decision: must(transactionId), duplicate: true };

  const deposit = await fetchDeposit(transactionId);
  if (!deposit) {
    // Nothing was sent, so release the claim and let the caller try again.
    setState(decision.id, ['submitting'], 'created');
    return { http: 503, decision: must(transactionId), duplicate };
  }
  if (deposit.verifyStatus !== 0) {
    if (deposit.verifyStatus === status) {
      setState(decision.id, ['submitting'], 'done');
      return { http: 200, decision: must(transactionId), duplicate };
    }
    if (deposit.verifyStatus === 2 || deposit.verifyStatus === 3) {
      setState(decision.id, ['submitting'], 'failed');
      return { http: 409, decision: must(transactionId), duplicate };
    }
    setState(decision.id, ['submitting'], 'needs_review');
    return { http: 202, decision: must(transactionId), duplicate };
  }

  db().prepare('UPDATE deposit_decisions SET submitted_at = ? WHERE id = ?').run(Date.now(), decision.id);
  const reply = await zibal.changeDepositStatus(config.accountId, transactionId, status);
  const result = resultOf(reply);
  const base = { last_result: result, last_message: typeof reply.json?.message === 'string' ? reply.json.message : null };

  let http: number;
  if (reply.http === 200 && reply.json?.success === true) {
    const tracker = reply.json.data?.refundRequestTrackerId;
    setState(decision.id, ['submitting'], 'done', { ...base, refund_tracker_id: typeof tracker === 'string' ? tracker : null });
    http = 200;
  } else if (isRejected(reply)) {
    setState(decision.id, ['submitting'], 'failed', base);
    http = 422;
  } else {
    setState(decision.id, ['submitting'], 'unknown', base);
    http = 202;
  }
  log('deposit_decided', { id: decision.id, http: reply.http, result, state: must(transactionId).state });
  return { http, decision: must(transactionId), duplicate };
}

export async function reconcileDeposit(transactionId: string): Promise<Decision | undefined> {
  const decision = find(transactionId);
  if (!decision) return decision;
  if (decision.state === 'submitting' && Date.now() - decision.updated_at >= config.timeoutMs * 3) {
    // The process died after claiming. The change-status call may or may not have been sent.
    setState(decision.id, ['submitting'], 'unknown');
  } else if (decision.state !== 'unknown') {
    return decision;
  }

  const deposit = await fetchDeposit(transactionId);
  if (deposit?.verifyStatus === decision.status) {
    setState(decision.id, ['unknown'], 'done');
  } else {
    const age = Date.now() - (decision.submitted_at ?? decision.created_at);
    // A changed status that is not ours, or no change after the window, needs a person.
    const changedElsewhere = deposit && deposit.verifyStatus !== 0;
    if (changedElsewhere || age >= config.reviewAfterMs) setState(decision.id, ['unknown'], 'needs_review');
  }
  return find(transactionId);
}

export async function reconcileDue(): Promise<number> {
  const rows = db().prepare(`
    SELECT transaction_id FROM deposit_decisions
    WHERE state = 'unknown' OR (state = 'submitting' AND updated_at <= ?)
    ORDER BY updated_at LIMIT 50
  `).all(Date.now() - config.timeoutMs * 3) as { transaction_id: string }[];
  for (const row of rows) await reconcileDeposit(row.transaction_id);
  return rows.length;
}
```

### `lib/http.ts`

```ts
import 'server-only';
import { createHash, timingSafeEqual } from 'node:crypto';
import { NextRequest, NextResponse } from 'next/server';
import { config } from './config';

const digest = (value: string) => createHash('sha256').update(value).digest();

export const INVALID = Symbol('invalid');

// Route handlers have no default body limit.
export async function readJson(request: NextRequest, limit: number): Promise<unknown> {
  if (!(request.headers.get('content-type') ?? '').startsWith('application/json')) return INVALID;
  if (Number(request.headers.get('content-length') ?? 0) > limit) return INVALID;
  const reader = request.body?.getReader();
  if (!reader) return INVALID;
  const chunks: Uint8Array[] = [];
  let size = 0;
  for (;;) {
    const { done, value } = await reader.read();
    if (done) break;
    size += value.length;
    if (size > limit) {
      await reader.cancel();
      return INVALID;
    }
    chunks.push(value);
  }
  try {
    return JSON.parse(Buffer.concat(chunks).toString('utf8'));
  } catch {
    return INVALID;
  }
}

export function json(body: unknown, status = 200) {
  return NextResponse.json(body, { status, headers: { 'Cache-Control': 'no-store' } });
}

export function unauthorized(request: NextRequest): NextResponse | null {
  const given = request.headers.get('x-internal-key') ?? '';
  return timingSafeEqual(digest(given), digest(config.internalKey)) ? null : json({ error: 'unauthorized' }, 401);
}
```

### `app/payouts/route.ts`

```ts
import { NextRequest } from 'next/server';
import { z } from 'zod';
import { INVALID, json, readJson, unauthorized } from '@/lib/http';
import { Conflict, createPayout, view } from '@/lib/payouts';

export const dynamic = 'force-dynamic';

// amount, iban and delay limits come from the docs. Other caps are this app's own choice.
const text = (max: number) => z.string().max(max).regex(/^[^\p{Cc}]*$/u);
const schema = z.strictObject({
  requestKey: z.string().regex(/^[A-Za-z0-9_-]{1,64}$/),
  amount: z.number().int().min(10000).max(Number.MAX_SAFE_INTEGER),
  iban: z.string().regex(/^IR[0-9]{24}$/),
  reasonCode: z.number().int().min(1).max(19),
  delay: z.number().int().min(-1).max(10080).optional(),
  description: text(255).optional(),
  paymentNumber: text(64).optional(),
  split: z.boolean().optional(),
});

function httpStatus(state: string) {
  if (state === 'failed') return 422;
  if (state === 'unknown' || state === 'needs_review' || state === 'submitting') return 202;
  return 201;
}

export async function POST(request: NextRequest) {
  const denied = unauthorized(request);
  if (denied) return denied;
  const body = await readJson(request, 16384);
  const parsed = body === INVALID ? null : schema.safeParse(body);
  if (!parsed?.success) return json({ error: 'invalid_request' }, 400);
  try {
    const { payout, duplicate } = await createPayout(parsed.data);
    return json(view(payout, { duplicate }), duplicate ? 200 : httpStatus(payout.state));
  } catch (error) {
    if (error instanceof Conflict) return json({ error: error.message }, 409);
    throw error;
  }
}
```

### `app/payouts/[id]/route.ts`

```ts
import { NextRequest } from 'next/server';
import { json, unauthorized } from '@/lib/http';
import { getPayout, view } from '@/lib/payouts';

export const dynamic = 'force-dynamic';

export async function GET(request: NextRequest, { params }: { params: Promise<{ id: string }> }) {
  const denied = unauthorized(request);
  if (denied) return denied;
  const payout = getPayout((await params).id);
  return payout ? json(view(payout)) : json({ error: 'not_found' }, 404);
}
```

### `app/payouts/[id]/reconcile/route.ts`

```ts
import { NextRequest } from 'next/server';
import { json, unauthorized } from '@/lib/http';
import { reconcilePayout, view } from '@/lib/payouts';

export const dynamic = 'force-dynamic';

export async function POST(request: NextRequest, { params }: { params: Promise<{ id: string }> }) {
  const denied = unauthorized(request);
  if (denied) return denied;
  const payout = await reconcilePayout((await params).id);
  return payout ? json(view(payout)) : json({ error: 'not_found' }, 404);
}
```

### `app/payouts/[id]/cancel/route.ts`

```ts
import { NextRequest } from 'next/server';
import { json, unauthorized } from '@/lib/http';
import { cancelPayout, Conflict, getPayout, view } from '@/lib/payouts';

export const dynamic = 'force-dynamic';

export async function POST(request: NextRequest, { params }: { params: Promise<{ id: string }> }) {
  const denied = unauthorized(request);
  if (denied) return denied;
  const payout = getPayout((await params).id);
  if (!payout) return json({ error: 'not_found' }, 404);
  try {
    const outcome = await cancelPayout(payout);
    return json(view(outcome.payout, {
      cancelOutcome: outcome.cancelOutcome,
      cancelled: outcome.cancelled,
      uncancellable: outcome.uncancellable,
    }), outcome.http);
  } catch (error) {
    if (error instanceof Conflict) return json({ error: error.message }, 409);
    throw error;
  }
}
```

### `app/internal/reconcile/route.ts`

```ts
import { NextRequest } from 'next/server';
import * as deposits from '@/lib/deposits';
import { json, unauthorized } from '@/lib/http';
import * as payouts from '@/lib/payouts';

export const dynamic = 'force-dynamic';

export async function POST(request: NextRequest) {
  const denied = unauthorized(request);
  if (denied) return denied;
  return json({ processed: (await payouts.reconcileDue()) + (await deposits.reconcileDue()) });
}
```

### `app/deposits/[transactionId]/decision/route.ts`

```ts
import { NextRequest } from 'next/server';
import { z } from 'zod';
import { decideDeposit, view } from '@/lib/deposits';
import { INVALID, json, readJson, unauthorized } from '@/lib/http';
import { Conflict } from '@/lib/payouts';

export const dynamic = 'force-dynamic';

const bodySchema = z.strictObject({ status: z.union([z.literal(2), z.literal(3)]) });
const idSchema = z.string().regex(/^[A-Za-z0-9_-]{1,64}$/);

export async function POST(request: NextRequest, { params }: { params: Promise<{ transactionId: string }> }) {
  const denied = unauthorized(request);
  if (denied) return denied;
  const id = idSchema.safeParse((await params).transactionId);
  const raw = await readJson(request, 16384);
  const body = raw === INVALID ? null : bodySchema.safeParse(raw);
  if (!id.success || !body?.success) return json({ error: 'invalid_request' }, 400);
  try {
    const { http, decision, duplicate } = await decideDeposit(id.data, body.data.status);
    return json(view(decision, { duplicate }), http);
  } catch (error) {
    if (error instanceof Conflict) return json({ error: error.message }, 409);
    throw error;
  }
}
```

### `app/deposits/[transactionId]/reconcile/route.ts`

```ts
import { NextRequest } from 'next/server';
import { reconcileDeposit, view } from '@/lib/deposits';
import { json, unauthorized } from '@/lib/http';

export const dynamic = 'force-dynamic';

export async function POST(request: NextRequest, { params }: { params: Promise<{ transactionId: string }> }) {
  const denied = unauthorized(request);
  if (denied) return denied;
  const decision = await reconcileDeposit((await params).transactionId);
  return decision ? json(view(decision)) : json({ error: 'not_found' }, 404);
}
```

### `app/balance/route.ts`

```ts
import { NextRequest } from 'next/server';
import { config } from '@/lib/config';
import { json, unauthorized } from '@/lib/http';
import { resultGroup, resultOf, zibal } from '@/lib/zibal';

export const dynamic = 'force-dynamic';

export async function GET(request: NextRequest) {
  const denied = unauthorized(request);
  if (denied) return denied;
  const reply = await zibal.balance(config.accountId);
  if (reply.http === null) return json({ error: 'zibal_unreachable' }, 504);
  // Balance has no result field on success.
  if (reply.http === 200 && Number.isSafeInteger(reply.json?.balance)) {
    return json({ balance: reply.json.balance, balanceUpdatedAt: reply.json.balanceUpdatedAt });
  }
  const result = resultOf(reply);
  return json({ error: 'zibal_error', lastResult: result, errorGroup: resultGroup(result) }, 502);
}
```

### `app/webhooks/zibal/checkout/route.ts`

```ts
import { NextRequest } from 'next/server';
import { db } from '@/lib/db';
import { INVALID, json, readJson } from '@/lib/http';
import { reconcilePayout } from '@/lib/payouts';

export const dynamic = 'force-dynamic';

// Public on purpose. Only the unique code is read, and the state comes from inquire.
export async function POST(request: NextRequest) {
  const body: any = await readJson(request, 65536);
  if (body === INVALID) return json({ error: 'invalid_request' }, 400);
  const code = body?.data?.unique_code ?? body?.data?.uniqueCode;
  if (typeof code === 'string' && code.length <= 64) {
    const row = db().prepare('SELECT id FROM payouts WHERE unique_code = ?').get(code) as { id: string } | undefined;
    if (row) await reconcilePayout(row.id);
  }
  return json({ ok: true });
}
```

## Run

```bash
npm install
cp .env.example .env.local
npm run build
npm start
```

Test against a local mock by setting `ZIBAL_BASE_URL`. There is no sandbox, so a real token would act on real accounts.

## Framework notes

- Never prefix a credential with `NEXT_PUBLIC_`. That copies it into the browser bundle.
- `import 'server-only'` makes the build fail if a client component imports these modules.
- `cache: 'no-store'` and `dynamic = 'force-dynamic'` keep `checkout/inquire` and balance from being served from a cache. A cached inquire would show stale money state.
- `next build` evaluates route modules without your variables, so `config` and `db()` are lazy.
- `params` is a Promise in route handlers.
- Route handlers have no body limit, so `readJson` applies one while reading.
- Route handlers have no CSRF protection, so the webhook needs no exemption. The Origin check on Server Actions does not apply to them. Do not create transfers from a Server Action unless it has your own authorization.
- Zibal answers only allowed IPs. Serverless hosts have changing outbound IPs, so they need a static egress IP, or result 7 appears.
- Serverless functions can be killed mid-call. The `submitting` state and the scheduled reconcile cover that case. Keep the function timeout above `ZIBAL_TIMEOUT_MS`.
