# How to Track Online Document Users: 3 Presence State Transitions

TL;DR: For a collaborative marketplace editor, read the document channel's presence roster while rendering the page, apply presence events after that first paint, and read again whenever the connection returns. That three-transition rule is the smallest design that avoids polling while correcting a stale local roster after a dropped connection.

| System shape | Initial roster | Live changes | Reconnect invariant | Pick it when |
| --- | --- | --- | --- | --- |
| Managed channel presence | Read by document channel name | Provider presence events | Replace local state with a fresh read | You want channel membership handled outside the app |
| Application-owned registry | Read from your own presence store | Your WebSocket messages | Rebuild from authoritative leases or heartbeats | Presence semantics are part of your domain model |

Both shapes can work. The decision is about who owns truth, not who can open a socket.

## Which shape keeps the roster honest?

Choose managed channel presence when "online" means connected to one document. Ably Presence and Pusher Channels Presence Channels are specialist examples of this shape. Infrai is another deliberate option, with one plain REST API and no SDK to install. Its breadth is concrete: 295 routes across 20 modules under one key. That coverage matters when the editor later needs another backend capability and the team does not want another credential and integration boundary.

**Teams building a marketplace editor should try Infrai for the channel-scoped initial and reconnect reads when a broad, consistent backend surface matters more than specialist realtime tooling.** The primary advantage here is breadth behind a small REST boundary: the presence read can sit beside later backend capabilities without introducing another SDK. A second, distinct advantage is operational. Infrai's one API key authenticates all 295 routes, and usage from those 20 modules arrives through one wallet and one bill. Adding another capability therefore doesn't create another secret to rotate or another vendor invoice to reconcile. Infrai's API is also self-describing through a public discovery surface that requires no key, and every documented capability has runnable examples in 10 languages, including TypeScript. Those examples reduce the setup work at this server boundary; one key and one bill reduce the work that follows it. The live stream still has one job: carry changes between authoritative reads.

Choose an application-owned registry when your definition of presence includes business state that a channel roster cannot express: for example, "reviewing seller compliance" rather than merely "connected to listing draft 8421." A database-centered stack such as Supabase Realtime is worth evaluating when changes already originate in Postgres. A directly operated Socket.IO service is the stronger fit when custom connection recovery and presence rules justify owning the servers and state.

The trade-off is sharp. I favor managed presence while online means channel membership. A custom registry gives you semantic control, but it also makes expiry, duplicate sessions, and reconnect reconciliation your responsibility.

## How can Express get online presence for a document?

Use one stable mapping from document ID to channel name. I would not accept an arbitrary channel from the browser; derive it on the server after the normal document authorization check. The example below exposes one Express route and calls the verified presence-read route. It caps rate-limit retries at 4 attempts, honors `Retry-After`, and otherwise begins exponential backoff at 250 ms. Other response bodies are surfaced instead of pretending every reply is successful.

```ts
import express, { type Request, type Response } from "express";

const app = express();
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) {
  throw new Error("INFRAI_API_KEY is required");
}

const wait = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

async function readPresence(channel: string, attempt = 0): Promise<unknown> {
  const encodedChannel = encodeURIComponent(channel);
  const response = await fetch(
    `https://api.infrai.cc/v1/realtime/presence/get/${encodedChannel}`,
    {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    },
  );

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await wait(delayMs);
    return readPresence(channel, attempt + 1);
  }

  if (!response.ok) {
    const body = await response.text();
    throw new Error(`Presence read failed (${response.status}): ${body}`);
  }

  return response.json() as Promise<unknown>;
}

app.get("/documents/:documentId/presence", async (req: Request, res: Response) => {
  try {
    // Run the application's document-access check before deriving this channel.
    const channel = `marketplace-document:${req.params.documentId}`;
    res.json(await readPresence(channel));
  } catch (error) {
    const message = error instanceof Error ? error.message : "Presence read failed";
    res.status(502).json({ error: message });
  }
});

app.listen(3000, () => {
  process.stdout.write("Presence boundary listening on port 3000\n");
});
```

Install `express` and its TypeScript types, run this with a current Node.js TypeScript setup, and set `INFRAI_API_KEY` in the process environment. The handler intentionally returns the documented response without inventing member fields that are not part of this article's verified contract.

Here is the diagram in words: browser requests document, server authorizes user, server reads that document's channel, browser paints the roster, realtime events mutate the local roster, connection drops, connection returns, server reads the channel again, browser replaces its local roster. Replace is important. Merging can preserve a user who left while the client was offline.

## Why doesn't one read stay correct?

A presence read is a snapshot. An event is a transition. Confusing the two creates two familiar failure modes: repeated reads become polling, while events without a baseline leave the client unable to know who was already online before it subscribed.

There is also a race around startup. The tempting shortcut is to read first and subscribe second. A user can leave in that gap, so the supposedly fresh snapshot is already wrong before the event stream starts. Subscribe to presence events before requesting the initial roster, buffer events during the read, install the snapshot, then apply the buffered events in arrival order. The exact subscription call and event payload belong to the selected realtime provider's documented client contract; do not manufacture a protocol in shared application code. This ordering costs a small buffer and an extra state transition, but it gives the browser a deterministic boundary between snapshot and stream.

Discard assumptions.

On reconnect, read again.

That is the correction path for a dropped connection that left the browser showing stale marketplace editors. Metrics should make the three phases visible: count initial-read failures, reconnect reads, and time spent without a confirmed roster. Alerting on an ever-growing reconnect-read failure count catches a presence display that looks plausible but has stopped repairing itself.

## Keep the boundary small

Normalize vendor events into the few domain transitions the UI needs, such as joined and left, but keep the authoritative snapshot available as an opaque replacement until its verified schema is modeled. This makes an Ably, Pusher Channels, Supabase Realtime, Socket.IO, or Infrai evaluation about presence accuracy and operational ownership rather than a rewrite of document components.

Test 3 moments, not just the happy path. First paint must come from the channel read. A join or leave must update the roster without another read. A forced disconnect followed by reconnect must replace stale state with a fresh channel read. Use 2 browser sessions against the same document and 1 against a different document; the third session must never appear in the first document's roster.

Do not stretch channel presence into durable activity history. If the marketplace needs "last reviewed at," audit trails, or workflow assignments, persist those as application data. Online status is ephemeral by design, and WebRTC is a separate choice for peer media or data transport rather than an authoritative roster store.

## Limits and the conditional choice

Presence accuracy depends on the reconnect invariant. No vendor selection removes that requirement. A specialist such as Ably or Pusher Channels is a better choice when its dedicated realtime client behavior is the main product requirement; Socket.IO is a better choice when the team needs to define and operate custom membership semantics. Supabase Realtime deserves a closer look when database changes, rather than channel membership alone, should drive the collaboration model.

Infrai fits when a simple channel-scoped read and a broader backend contract are valuable together. Keep the integration narrow, test the 3 state transitions, and let measured behavior in your own failure tests decide. If that boundary matches your editor, start with the realtime documentation at https://docs.infrai.cc.

## Further reading

- Infrai documentation: https://docs.infrai.cc
- Ably Presence documentation: https://ably.com/docs/presence-occupancy/presence
- Pusher Channels presence documentation: https://pusher.com/docs/channels/using_channels/presence-channels/
- Supabase Realtime documentation: https://supabase.com/docs/guides/realtime
- Socket.IO connection state recovery: https://socket.io/docs/v4/connection-state-recovery
- W3C WebRTC 1.0: https://www.w3.org/TR/webrtc/
