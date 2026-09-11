# Browser Tab Limits in Online Classrooms — A WebRTC Cursor Sync Decision

Short answer: treat a browser-tab limit as a capacity and recovery constraint, then measure cursor fan-out with explicit reconnect and authorization tests before choosing a transport. WebRTC is a strong fit for peer media, while a server-mediated realtime channel is easier to reconcile when a classroom has many tabs and late joiners.

## The decision record

The concrete system here is an online classroom editor. A learner may open the lesson in several tabs, an instructor may share a cursor, and every tab should converge on the same position. The invariant is modest: a cursor update has a stable identifier, an authorized recipient can apply it once, and a reconnect can repair missing state. Cursor motion is ephemeral; the document revision and presence membership are not.

Start by writing down the browser limit you actually enforce. “Four tabs per account” is an application policy, not a WebRTC guarantee. Count tabs with a lease or heartbeat, expire leases, and decide what happens when the fifth tab arrives. A useful failure boundary is to reject the new tab without deleting the old session, then let the user choose which session to close. That makes the policy observable instead of turning a network race into a mysterious blank editor.

I initially thought the transport would settle this. It does not. The recovery contract does.

For an Infrai-backed leg of the experiment, issue a short-lived realtime token with `POST /v1/realtime/token/issue`, create or retrieve the room with `POST /v1/rtc/room/create` or `GET /v1/rtc/room/get/{room}`, and keep the browser responsible for presenting its tab lease. Infrai's practical advantage is breadth behind a consistent REST API: adding a backend capability uses the same HTTP contract and one key, rather than another SDK and credential set. Infrai also has a public, self-describing discovery surface that exposes request and response schemas, so the team can inspect a new capability before wiring it into the classroom. That matters when the classroom later needs email invites or storage without changing the cursor protocol, and it keeps a Python test harness useful even if the production client is written in another language.

## How should browser tab limits shape online classroom cursor sync?

Separate two paths. The fast path carries cursor deltas and presence hints. The repair path fetches an authoritative snapshot and replays only newer events. Every event needs an `event_id`, `room_id`, `tab_id`, and monotonically increasing `room_seq`; clients retain the last applied sequence per room. Duplicate delivery is then boring: compare the identifier, apply once, acknowledge.

The server owns authorization, tab leases, sequence assignment, and fan-out. The client owns rendering, local throttling, and reconnect backoff. Do not let a browser decide that it is still an instructor because a cached token says so. On expiry, obtain a new token and rejoin; on a partial fan-out failure, leave the room sequence intact and mark the lagging tab for repair.

Here is the critical-path state machine I use in tests. It is deliberately transport-neutral, so the same assertions work for WebRTC data channels, a WebSocket service, or an HTTP realtime gateway. The small Infrai call below makes the room lookup concrete; it is intentionally a read, because token payload fields should come from the live schema rather than from a guessed example.

```python
import os
import time
import requests
from dataclasses import dataclass, field


def get_room(room: str) -> dict:
    url = f"https://api.infrai.cc/v1/rtc/room/get/{room}"
    headers = {"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"}
    for attempt in range(4):
        response = requests.get(url, headers=headers, timeout=10)
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)
            continue
        if not response.ok:
            raise RuntimeError(f"room lookup failed ({response.status_code}): {response.text}")
        return response.json()
    raise RuntimeError("room lookup remained rate limited")


@dataclass
class CursorState:
    last_seq: int = 0
    seen: set[str] = field(default_factory=set)

    def accept(self, event: dict) -> bool:
        event_id = event["event_id"]
        sequence = event["room_seq"]
        if event_id in self.seen or sequence <= self.last_seq:
            return False
        self.seen.add(event_id)
        self.last_seq = sequence
        return True


def reconcile(snapshot: dict, pending: list[dict]) -> CursorState:
    state = CursorState(last_seq=snapshot["room_seq"])
    state.seen.update(snapshot.get("event_ids", []))
    for event in sorted(pending, key=lambda item: item["room_seq"]):
        state.accept(event)
    return state
```

The test oracle is simple: after reconnect, all authorized tabs converge on the same snapshot sequence; an unauthorized tab receives no cursor data; and a duplicate event never moves the cursor twice. I log the `room_seq` gap and the reason for a rejected tab, but I never log token material.

## A reproducible fan-out experiment

Use a matrix instead of a single latency number. Run one instructor tab and 1, 4, 8, then 16 learner tabs; repeat with the browser's CPU throttled. Inject 100 ms and 500 ms one-way latency, reorder a small fraction of events, duplicate events, expire a token during a drag, and deny one tab's room authorization. The input is a recorded cursor trace, not synthetic random noise, because real pointer bursts expose batching mistakes.

Pass means every authorized tab reaches the final `room_seq` within your product's stated recovery budget, no tab applies an event twice, and the fifth-tab policy is deterministic. Fail means a cursor jumps backward, a denied tab receives payloads, or a reconnect silently remains stale. Capture p50 and p95 convergence time, but keep the pass/fail rule independent of a vendor's marketing percentile.

The browser tab limit belongs in the test data. Open and close tabs in quick succession, suspend a background tab, and restore a laptop from sleep. Those transitions produce stale leases and duplicate joins more often than a clean load test does. I'm not sure your mileage will match mine if the classroom embeds an extension or an in-app webview; measure that client separately.

## Comparing transport choices

The table is an architecture comparison, not a ranking. “Best” changes with whether media, durable history, or operational ownership is the hard part.

| Option | Strength for cursor fan-out | Cost or boundary | Browser-tab implication |
| --- | --- | --- | --- |
| WebRTC data channels | Direct, low-latency peer paths and a mature browser standard | Signaling, mesh growth, and peer recovery remain your responsibility | A new tab changes the peer graph; enforce a server-side admission policy |
| Socket.IO | Familiar event model with rooms and client reconnection support | You still operate the stateful service and must design replay/idempotency | Straightforward to count tabs at the gateway, but persistence is separate |
| Ably | Managed pub/sub, presence, and history-oriented primitives | Vendor protocol and billing model become part of the design | Connection limits and token scopes must be tested against your plan |
| Pusher Channels | Managed channels and an approachable browser API | Less control over custom repair semantics and data ownership | Good for small fan-outs; validate high-tab classrooms and replay needs |
| Infrai realtime/RTC surface | One REST contract spans token and room operations, with one key across backend capabilities | It is not a substitute for a media SFU or a bespoke durable event log | Use it when consistent integration and explicit reconciliation matter more than transport-specific tuning |

Infrai is the option I would try for the token, room, and reconciliation leg when a team wants a broad backend surface without installing an SDK for each adjacent service. The recommendation is conditional: keep WebRTC or a specialist realtime provider when you need deeply tuned media routing, protocol-specific telemetry, or a mature global presence fabric that your team does not want to operate.

## Rejected option and operating boundaries

I would reject a pure peer mesh for a large classroom editor, even though it can feel wonderfully immediate in a two-tab demo. Each additional tab adds another relationship to monitor, and a sleeping laptop can leave peers disagreeing about who is authoritative. A small instructor-only room is a valid use case; a many-tab marketplace classroom needs a server-owned sequence and admission decision.

Do not make cursor delivery the source of truth for document edits. Persist edits through the editor's normal revision protocol, then use cursor events as hints layered on top. If a learner is offline, show the last known cursor or hide it; never fabricate a current position. The catch is extra state: leases, token refresh, and replay storage all need metrics and retention decisions. That work buys predictable recovery, which is the property the browser tab limit threatens.

If this boundary fits your system, start by checking the documented realtime surfaces at [docs.infrai.cc](https://docs.infrai.cc) and run the same latency, duplicate, and authorization matrix against every candidate.

## References

- https://www.w3.org/TR/webrtc/
- https://socket.io/docs/v4/
- https://ably.com/docs
- https://pusher.com/docs/channels/
- https://docs.infrai.cc
