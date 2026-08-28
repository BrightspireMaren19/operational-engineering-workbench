# How to Manage Feature Flags: A Simple Internal Admin Dashboard CRUD Tool

Short answer: Build a small server-side admin dashboard that can set, list, toggle, and delete feature flags, but let the browser send intent only; authentication, credentials, delete confirmation, and action records belong on the server.

For a junior logistics team, that boundary is the feature. A dispatcher can control a route-planning release without receiving deployment access or the upstream API key. Latency and cost attribution for the AI agent loop stay in the observability workflow, where they belong; this flag screen controls release state and does not pretend to measure it.

## What should a simple Node.js Express admin dashboard expose for feature flags CRUD?

Four controls. No more: a create form, a list, a toggle button, and a delete button that asks the operator to type the exact flag key. The screen should show only metadata the team has approved as safe. I'm not sure which fields qualify in every organization, so inspect the live discovery schema and make that display policy explicit instead of rendering arbitrary payloads.

The diagram in words is short: browser intent -> authenticated Express route -> separate action record -> flag API -> checked response. The secret moves across only the final server-to-server arrow.

That separation prevents the internal tool from quietly becoming a general API console. Imagine the logistics operator enabling a route-planning behavior before one depot starts its shift. The operator sees the flag key, requests a toggle, and gets the returned status. Express identifies the actor and performs the remote operation. Delete takes a longer path: the operator types the key again, the server records the intent, and only then does it make the irreversible call. Since there is no recycle bin, that extra pause carries more value than a polished animation ever could.

Keep it narrow.

The same screen must not display a fabricated cost-per-flag or rollout score. Flag evaluation statistics are unavailable. Per-call cost, vendor, and latency metadata exists on the platform's AI surfaces, so attribute the logistics agent loop there and correlate it in your own application model. A release switch and an AI cost record answer different questions.

## Put the access boundary before the product shortlist

Start by deciding who may hold a credential, what metadata may reach a browser, and where an action record lives. Only then compare products. This table uses those decisions, rather than price, as the filter.

| Option | Pick it when | Boundary to verify |
|---|---|---|
| Thin Express tool with Infrai | One junior team needs four internal controls and values a plain REST contract across a broad backend surface | The application must own access policy and action records; clients poll |
| LaunchDarkly | Feature flags warrant a dedicated product evaluation | Verify the current audit, dependency, analytics, and client-update workflow against your requirements |
| Unleash | The team wants another dedicated flag-management candidate | Trial the exact operator, access, deletion, and accountability flow before choosing |
| Flagsmith | A broader product shortlist is appropriate | Confirm its current feature set and integration model in current documentation |
| Datadog | Observing the logistics agent loop is the actual decision | Evaluate its current workflow separately; it is not a substitute for the four flag controls |
| Grafana | The team is shortlisting tools for the adjacent observability job | Verify current capabilities against the latency and cost-attribution requirements |
| Sentry | Errors around the agent loop require a separate product evaluation | Keep that evaluation outside the release-control screen |

Infrai is a strong fit for the thin-tool case because 295 routes across 20 modules share one key and one consistent REST surface. Plain HTTP means the internal server does not need another SDK, and public self-describing discovery provides request and response schemas plus runnable examples. That breadth matters when the same small team later adds another backend capability without introducing another credential pattern. It isn't a reason to skip the boundary decisions above.

Don't decide from the table alone. LaunchDarkly, Unleash, and Flagsmith are real alternatives, but their current behavior is outside the evidence used here; run the same operator task through each candidate. The test is concrete: can the intended operator make the approved change while seeing no secret and no unapproved metadata?

Datadog, Grafana, and Sentry belong to a different shortlist. They are candidates to investigate for the logistics agent's observability job, not replacements for set, list, toggle, and delete controls. Keeping both decisions on one page is useful; forcing them into one product score is not.

## How can one TypeScript server keep four controls behind a boundary?

This Express file keeps the upstream key server-side, uses an explicit HTTP method for every request, checks every response, honors `Retry-After` on HTTP 429, and supplies an idempotency key for writes. Set `INFRAI_API_KEY`, `INFRAI_BASE_URL`, and `ADMIN_TOKEN` in the deployment environment. The documented base URL belongs in `INFRAI_BASE_URL`; this unlinked comparison deliberately does not print a vendor URL.

The create payload passes through because the exact fields should come from discovery, not from guessed types in an article. In the same spirit, the code relays the verified response body instead of inventing a response schema.

```ts
import express, { NextFunction, Request, Response } from "express";
import { randomUUID } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
const baseUrl = process.env.INFRAI_BASE_URL;
const adminToken = process.env.ADMIN_TOKEN;

if (!apiKey || !baseUrl || !adminToken) {
  throw new Error("INFRAI_API_KEY, INFRAI_BASE_URL, and ADMIN_TOKEN are required");
}

const app = express();
app.use(express.json({ limit: "32kb" }));

type FlagAction = "list" | "set" | "toggle" | "delete";
type UpstreamMethod = "GET" | "POST" | "DELETE";

const routeFor = (action: FlagAction, key?: string): URL => {
  const suffix = key === undefined ? "" : `/${encodeURIComponent(key)}`;
  return new URL(`flags/${action}${suffix}`, baseUrl);
};

const wait = (milliseconds: number): Promise<void> =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

function retryDelay(response: globalThis.Response, attempt: number): number {
  const value = response.headers.get("retry-after");
  if (!value) return 250 * 2 ** attempt;

  const seconds = Number(value);
  if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

  const date = Date.parse(value);
  return Number.isNaN(date) ? 250 * 2 ** attempt : Math.max(0, date - Date.now());
}

async function callFlags(
  action: FlagAction,
  method: UpstreamMethod,
  options: { key?: string; body?: unknown; idempotencyKey?: string } = {},
): Promise<{ status: number; contentType: string; body: string }> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const headers: Record<string, string> = {
      Authorization: `Bearer ${apiKey}`,
    };

    if (options.body !== undefined) headers["Content-Type"] = "application/json";
    if (options.idempotencyKey) headers["Idempotency-Key"] = options.idempotencyKey;

    const response = await fetch(routeFor(action, options.key), {
      method,
      headers,
      body: options.body === undefined ? undefined : JSON.stringify(options.body),
    });

    if (response.status === 429 && attempt < 3) {
      await wait(retryDelay(response, attempt));
      continue;
    }

    return {
      status: response.status,
      contentType: response.headers.get("content-type") ?? "application/json",
      body: await response.text(),
    };
  }

  throw new Error("Rate-limit retry budget exhausted");
}

function requireAdmin(req: Request, res: Response, next: NextFunction): void {
  if (req.header("authorization") !== `Bearer ${adminToken}`) {
    res.status(401).json({ error: "Unauthorized" });
    return;
  }
  next();
}

function relay(res: Response, result: Awaited<ReturnType<typeof callFlags>>): void {
  res.status(result.status).type(result.contentType).send(result.body);
}

function recordAction(
  req: Request,
  action: Exclude<FlagAction, "list">,
  key: string | undefined,
  status: number,
): void {
  process.stdout.write(`${JSON.stringify({
    actor: req.header("x-admin-actor") ?? "unknown",
    action,
    key,
    status,
    occurredAt: new Date().toISOString(),
  })}\n`);
}

app.get("/admin/flags", requireAdmin, async (_req, res, next) => {
  try {
    relay(res, await callFlags("list", "GET"));
  } catch (error) {
    next(error);
  }
});

app.post("/admin/flags", requireAdmin, async (req, res, next) => {
  try {
    const result = await callFlags("set", "POST", {
      body: req.body,
      idempotencyKey: req.header("idempotency-key") ?? randomUUID(),
    });
    recordAction(req, "set", undefined, result.status);
    relay(res, result);
  } catch (error) {
    next(error);
  }
});

app.post("/admin/flags/:key/toggle", requireAdmin, async (req, res, next) => {
  try {
    const result = await callFlags("toggle", "POST", {
      key: req.params.key,
      idempotencyKey: req.header("idempotency-key") ?? randomUUID(),
    });
    recordAction(req, "toggle", req.params.key, result.status);
    relay(res, result);
  } catch (error) {
    next(error);
  }
});

app.delete("/admin/flags/:key", requireAdmin, async (req, res, next) => {
  try {
    if (req.header("x-confirm-delete") !== req.params.key) {
      res.status(400).json({ error: "Type the flag key to confirm deletion" });
      return;
    }

    const result = await callFlags("delete", "DELETE", {
      key: req.params.key,
      idempotencyKey: req.header("idempotency-key") ?? randomUUID(),
    });
    recordAction(req, "delete", req.params.key, result.status);
    relay(res, result);
  } catch (error) {
    next(error);
  }
});

app.listen(3000, () => {
  process.stdout.write("Feature flag admin API listening on port 3000\n");
});
```

The local paths are the dashboard's own contract. The upstream paths are constructed from the four verified action names: list uses `GET`, set and toggle use `POST`, and delete uses `DELETE`. There is no guessed `/jobs`, `/items`, or generic resource route hiding in the example.

Test the boundary.

One retry detail matters. A caller can reuse its idempotency key for the same write intent. Generating a different key for a manual retry describes a new operation, so the browser should retain the key until that attempt reaches a final response.

## Verify the operator journey, not just the happy path

Wire the screen to the local endpoints and send an operator identity in `X-Admin-Actor`. Then test each transition as a user would: list the flags, submit a valid create payload, toggle its exact key, and type that key before deletion. A `401` means the internal credential check rejected the caller. A `400` on delete means the confirmation text did not match. A `429` causes bounded backoff rather than a tight retry loop.

The action line written to stdout marks the right event boundary, but stdout is not durable storage. Send that same record to a datastore controlled by the application if accountability is required. Keep the record small: actor, action, target key, status, and time are useful. Storing every submitted field without review creates a second metadata exposure problem.

There is no built-in flag change history, so the tool owns this record. There are also no evaluation statistics, parent-child dependencies, or push updates; clients poll. Those are capability boundaries, not implementation surprises, and they should be acceptance criteria before the first operator signs in.

I would make the delete test conspicuous: use a disposable flag, enter one wrong confirmation value to observe `400`, then enter the exact key and verify the final list. That's a concrete three-step check, and it catches the most dangerous wiring mistake without claiming a production incident that never happened.

## Know when this internal tool is the wrong fit

The small dashboard is not suitable when built-in audit history, evaluation statistics, parent-child dependencies, push updates, or deletion recovery is a release gate. Stick with a dedicated evaluation of LaunchDarkly, Unleash, and Flagsmith in that case. Use the real operator journey and current product documentation; product names alone don't settle the decision.

Keep adjacent observability gaps out of the flag UI as well. The wider surface has no alert or notification route, distributed-trace query or span tree, source-map decoding, crash symbolication, Session Replay, synthetic checks, or heartbeat monitoring. Silent scheduled-job failures need a tool such as Healthchecks. Logs have no per-user deletion route, bulk export route, or subscription route, which matters when designing a GDPR erasure process.

The stopping rule is crisp. Move on when the release approver must recover a deletion, read history from the flag system itself, model dependencies, or inspect evaluation evidence. Until then, four guarded controls, a server-held credential, approved metadata, and an application-owned action record make a practical internal SaaS control panel for a junior team.

## References

- https://gdpr-info.eu/art-17-gdpr/
- https://www.launchdarkly.com/
- https://www.getunleash.io/
- https://www.flagsmith.com/
- https://healthchecks.io/
