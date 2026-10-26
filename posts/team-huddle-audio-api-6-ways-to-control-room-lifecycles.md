# Team Huddle Audio API: 6 Ways to Control Room Lifecycles

For a team huddle audio room API, give each clinician a short-lived, room-scoped join grant, and treat online presence as a server-owned lease rather than proof that a person is listening. The deciding constraint is client trust: a browser may request access and report liveness, but it must never choose its workspace, role, room, or cleanup deadline.

**Short answer:** separate the durable huddle record, the expiring presence lease, and the media session. This keeps a shared clinical workspace useful during reconnects without turning a stale tab into an authority source. The six decisions below form the architecture decision record for an ad hoc standup huddle.

## 1. What Should a Team Huddle Audio Room API Trust?

The invariant is narrow: the backend is authoritative for membership and room phase; the client is authoritative only for local device intent. A nurse clicking Join may express intent to enter `ward-rounds`, but the server derives the clinician ID, workspace ID, and permitted role from the authenticated session. It then issues a grant restricted to that one room.

A green dot should mean "the server has received a valid renewal within the lease window." It should not mean "this person heard the last sentence." WebRTC defines browser-facing real-time communication interfaces, including peer connections and media tracks, but the application still owns its authorization and business lifecycle.

Keep those meanings separate.

That boundary holds.

## 2. Make six decisions explicit

1. Derive scope on the server. Never accept a workspace or clinical role merely because it appears in a request body. Compare the requested huddle against authenticated membership, then mint a single-room grant.
2. Lease presence. Renew an online marker every 15 seconds and expire it after 45 seconds in this design. Those values are policy choices, not protocol requirements; the three-renewal window absorbs one delayed heartbeat without leaving abandoned tabs visible for long.
3. Model room phases. Use `pending`, `active`, `draining`, and `closed`. A room becomes active after the first accepted join, enters draining after its last lease expires, and closes only after a server timer confirms that it is still empty.
4. Make joins idempotent. A retry carrying the same join nonce returns the same logical participation record. It must not increment occupancy twice or create two grants.
5. Separate signaling from media. Signaling carries offers, answers, and connectivity candidates; media flows through the selected WebRTC topology. Authorization gates both, but a signaling socket is not itself evidence that audio is flowing.
6. Observe transitions, not microphone content. Record join acceptance, renewal lag, reconnect count, rejected scope, and room-phase changes. Avoid putting patient names, clinical notes, SDP bodies, or connectivity candidates into routine logs.

The 15/45-second choice is deliberately visible because hidden timing creates support mysteries. A UI may remain optimistic during one missed renewal, while the server's lease clock remains final.

Consider one concrete edge sequence. A clinician joins the morning standup, changes networks, and reconnects while the final heartbeat from the old connection is still inside its 45-second lease. The backend should bind the new transport to the existing logical participation, reject any attempt to broaden the room scope, and keep one occupancy record. If the old transport later reports a disconnect, that late event must not delete the newer lease. A generation number or compare-and-set on the participation record makes the ordering explicit. Meanwhile, the room stays active because at least one valid lease remains. This is why a disconnect callback is an observation, not a cleanup command: callbacks can arrive after the state they describe has already been replaced.

## 3. Compare the lifecycle options once

| Option | Trust boundary | Empty-room behavior | Best fit | Main cost |
|---|---|---|---|---|
| Client-declared online state | Browser chooses presence | Last client announces closure | Disposable, low-impact rooms | Stale or malicious state can outlive reality |
| Socket-bound state | Signaling connection implies presence | Close after every socket disconnects | Stable networks and simple sessions | Reconnects can create visible flapping |
| Server lease plus room state machine | Backend validates scope and renewals | Drain, recheck, then close | Clinical workspaces with brief network gaps | More timers and transition tests |

The decision is the third row. It gives the UI a bounded claim and gives operations an auditable state transition. It also makes the failure boundary plain: loss of signaling eventually removes presence, while loss of a media track can be shown separately as an audio status problem.

## 4. Put the critical path behind one transaction

The public application may be implemented in Node.js, but the contract matters more than the runtime. The following Python expresses the critical path without tying the design to a commercial SDK. `authorize_member`, `rooms`, and `grants` are server-owned interfaces; clients cannot write their results.

```python
from dataclasses import dataclass
from datetime import datetime, timedelta, timezone

LEASE_SECONDS = 45

@dataclass(frozen=True)
class JoinRequest:
    room_id: str
    join_nonce: str

def join_huddle(actor, request, rooms, grants):
    now = datetime.now(timezone.utc)
    room = rooms.lock(request.room_id)

    # Membership and role come from the authenticated actor, not client claims.
    membership = rooms.authorize_member(
        actor_id=actor.id, workspace_id=room.workspace_id
    )
    if room.phase == "closed":
        raise PermissionError("room is closed")

    existing = grants.find_by_nonce(actor.id, room.id, request.join_nonce)
    if existing is not None:
        return existing

    participation = grants.create(
        actor_id=actor.id,
        room_id=room.id,
        role=membership.role,
        expires_at=now + timedelta(seconds=LEASE_SECONDS),
    )
    rooms.activate_if_pending(room.id, now)
    return participation
```

A production implementation should commit nonce uniqueness, participation creation, and the phase change atomically. Test the awkward edges: two simultaneous joins with one nonce, renewal arriving while a drain timer fires, authorization revoked between reconnect attempts, and a closed room referenced by a cached tab. Assertions should target final state, not timer ordering.

Deployment deserves the same discipline. Roll out a new room-phase transition with readers that understand both old and new records, then update writers. Alert on rising join rejection, renewal lateness, rooms stuck in draining, and duplicate-nonce conflicts. Do not page merely because one clinician closes a laptop.

## 5. Where Does the Failure Boundary Sit, and When Is This Unsuitable?

There are three independent boundaries. Authentication failure prevents grant creation. Signaling failure stops negotiation or renewal and eventually lets the presence lease expire. Media failure changes audio status but does not authorize a new participant or rewrite room membership.

This distinction matters during an on-call standup. If the microphone permission is denied, the clinician may remain visibly online but marked unable to publish audio. If connectivity restarts, the existing logical participation can survive while new signaling material is exchanged, provided the room grant remains valid. The server must recheck scope when refreshing that grant; possession of an old browser object is insufficient.

Do not infer attendance, care delivery, or acknowledgement from any of these states. They are operational signals for a shared workspace, nothing more. Compliance language should match that limited claim, and retention should be set for operational need rather than convenience.

The rejected design is "close the room as soon as the last signaling socket disconnects." It is attractive because it has fewer records, fewer timers, and an easy mental model. It is also a poor match for mobile clinicians moving between Wi-Fi and cellular coverage: a transport event becomes a business event, so a brief reconnect can end the huddle for everyone.

The limitation is operational complexity: leases require a shared clock policy, durable transition logic, and race tests. This approach is not suitable when a disposable room has no reconnect continuity, auditability, or stable-presence requirement; socket-bound cleanup is then understandable and cheaper to operate. That trade-off is valid. For the healthtech workspace described here, server leases and an explicit draining phase better contain client trust while preserving ad hoc room behavior.

## References

- [WebRTC 1.0: Real-Time Communication Between Browsers](https://www.w3.org/TR/webrtc/)
