---
name: multiplayer-netcode
description: Design and build real-time multiplayer netcode. Chooses a topology (P2P, host-authoritative, dedicated-authoritative, relay) and a transport, then implements the latency-hiding stack - client prediction, server reconciliation, interpolation, extrapolation, lag compensation - or a deterministic lockstep/rollback simulation, with the mobile, cost and determinism traps called out. Use for "make my game multiplayer", "PvP", "netcode", "rollback", "desync", "rubber-banding", "players say their hits don't register".
---

# Real-Time Multiplayer Netcode

Distilled from **"Multiplayer Oyun Geliştirirken Bilmeniz Gerekenler"**, the
three-part series by [Oğuzhan Yılmaz](https://blog.oguzhan.info) — netcode on
*Headball Online 2* and *Goalbattle*. The engineering judgement and the two
post-mortems in §10 are his; the wording here is ours. Sources, part-by-part
map and further reading: [`references.md`](references.md).

Engine-agnostic, mobile-leaning. The one sentence to keep: **every technique
below buys smoothness with a lie, and every lie has to be paid back
somewhere** — in server cost, device cost, or a bug report from a player who
noticed.

## Reading the symptom (the vocabulary, and what it costs you)

Players report feelings, not measurements. Map the complaint before reaching
for a technique:

| Term | What it is | What you will hear | Where the fix lives |
|---|---|---|---|
| **Latency / lag** | delay between an action and its visible effect | "the game feels delayed" | §5 prediction |
| **Ping** | ICMP round trip — the *network* layer only | the number players quote at you | §8 placement and routing |
| **RTT** | round trip **including** client and server processing | RTT far above ping | §4 tick budget, your own handlers |
| **Jitter** | variance in arrival time; packets also reorder | stutter and rubber-banding at a "good" ping | §5.3 interpolation buffer |
| **Packet loss** | the packet never arrives at all | teleports, inputs that never register | §3 delivery classes, §5.2 replay |

Two consequences worth keeping. **RTT minus ping is the part you own** — your
serialization, your tick wait, your handlers — and it is the only part you can
shorten by writing better code. And **jitter hurts more than latency**: a
steady 120 ms plays better than a 60 ms that swings, because prediction can
hide a constant and cannot hide a surprise. Loss has many causes you will
never see (a Wi-Fi to cellular handover, a saturated uplink, a route change, a
server pegged at 100% CPU), so treat it as weather, not as a bug to eliminate.

## 0 · Discovery (ask first, max 4 questions)

1. **Genre and simultaneity** — turn-based, async, or real-time? How many
   players share one simulation, and how fast must an input show up on
   someone else's screen? (Turn-based: skip to §3 and use WebSocket. You are
   done.)
2. **Who is allowed to be wrong** — is cheating a business risk (ranked
   ladder, money, leaderboards)? That single answer picks the authority, and
   the authority picks everything else.
3. **Where you can afford to pay** — your servers (CPU + egress, forever) or
   the players' devices (determinism engineering + the weakest phone you
   support)? See §10; you do not get a third option.
4. **Can the simulation be bit-identical?** Engine physics and `float` say
   no. If rollback or lockstep is on the table, this is a build-vs-abandon
   decision, not a detail — decide it now.

## 1 · Write `docs/netcode.md` before you open a socket

One page, committed, naming the decisions: topology, authority, transport and
delivery classes, tick rate, snapshot/input format, buffer sizes
(interpolation delay, input ring, state history), how desync is detected, and
the cost number that justified the choice. Every later argument gets settled
by this file instead of by opinion.

## 2 · Topology — the first and most expensive decision

| Topology | Authority | Server cost | Cheat resistance | Fits |
|---|---|---|---|---|
| **P2P mesh** | shared / none | rendezvous only | lowest | small co-op, party games |
| **P2P host-authoritative** | one elected peer | rendezvous only | low — host advantage is real | console/party, friend lobbies |
| **Dedicated authoritative** | server | highest | highest | competitive FPS, anything with money |
| **Dedicated relay** | clients (deterministic) | low | medium — needs desync detection | 1v1 / 2v2 mobile PvP |

- **P2P is not free.** NAT hole punching needs a rendezvous server to swap
  each peer's observed IP/port, and a relay fallback for the pairs that never
  punch through. Budget both, and expect a single-digit percentage of pairs
  to need the fallback.
- **P2P traffic grows with the square of the lobby.** Every peer sends to
  every other peer, so latency and battery get worse with each player who
  joins. Cap the lobby size; do not discover the cap in production.
- **P2P code is not simpler.** One socket becomes an array of sockets, plus
  peer identity, join/leave, and who is authoritative right now.
- **A host-authoritative match dies when the host leaves**, and the host's
  hardware sets everyone's tick rate. Design host migration up front or state
  plainly in `docs/netcode.md` that you accept the ending.
- Never place authority where lying pays. If the client owns the score, the
  score is a suggestion.

## 3 · Transport — three delivery classes, not one protocol

- **Lobby, matchmaking, purchases, chat, match start/end:** TCP (HTTP or
  WebSocket). Reliable, ordered, TLS in one line — pay its three-way
  handshake once at connect and move on.
- **Live simulation state:** UDP. TCP's in-order guarantee is the wrong
  guarantee for a game — one lost packet stalls every packet behind it
  (head-of-line blocking), and its retransmit hands you a position you
  stopped caring about two ticks ago. That protocol-level stall *is* jitter,
  and no amount of client-side smoothing removes it.
- Build exactly **three delivery classes** over UDP and stop:
  - *unreliable, newest-wins* — positions, snapshots (drop the stale one)
  - *reliable, unordered* — one-off events (a goal, a pickup)
  - *reliable, ordered* — the few things that need a sequence
- **Do not write the reliability layer yourself first.** Reliable-UDP
  libraries (KCP-class, ENet-class, or your engine's transport) already
  solve acks, ordering and congestion; write your own only once you have
  measured a reason. And keep the classes honest — a reliable-ordered
  channel carrying positions reproduces TCP's stall one layer up.
- **Ship a TCP/WebSocket fallback.** Some carriers and corporate networks
  eat UDP. Measure the fallback rate as a first-class metric — it is always
  higher than you expect.
- **Choose the port defensively.** Pick from an unassigned range (7000–9000
  has room; engines commonly sit near 7777) and stay away from anything a
  router might recognise. A well-known port number earns you QoS rules, port
  mapping, rate limits, or a nationwide ISP block during someone else's
  malware incident — and you will spend days debugging your own code first.

## 4 · Rates and the bill

- **Frame rate is rendering (GPU, client). Tick rate is simulation (CPU,
  wherever authority lives).** They are independent: a 120 FPS client against
  a 30 Hz server is normal and correct.
- Pick the tick rate from the game's decision granularity, not from ambition.
  Fighting/shooter ≈ 60, sports/MOBA ≈ 20–30, slow co-op ≈ 10–20.
- **Compute the monthly bill before committing to a tick rate.** Cost scales
  with `tick × players × payload × concurrent matches`, in both CPU and
  egress. This number, not the netcode, is what kills studios.
- Serialization and encryption are real server CPU. Bit-pack, quantize
  (positions to fixed precision, angles to a byte or two), delta against the
  last acknowledged snapshot, and send each peer only what it can perceive
  (interest management). Never send what the receiver can derive.

## 5 · Latency hiding, server-authoritative — implement in this order

1. **Client prediction.** Apply the local input immediately *and* send it.
   The local player never waits a round trip to move. Without this, a 100 ms
   ping means 100 ms between the button and the character, which is a reason
   to uninstall.
2. **Server reconciliation.** Number every input. Each snapshot carries the
   last input sequence the server processed. The client adopts the server
   state and **replays every unacknowledged input** on top of it. Keep an
   input ring buffer of at least `max_RTT × tick_rate` entries.
   - Correct *quietly*: hard-snap only past an error threshold; below it,
     blend over a few frames. Every visible correction arrives as a
     "my character teleported" report.
3. **Interpolation for everyone else.** Render remote entities slightly in
   the past, behind the newest snapshot by ~2 snapshot intervals plus a
   jitter margin. This is what makes a 20 Hz server look smooth at 60 FPS,
   and the buffer is what absorbs jitter and reordering. Tune the delay per
   game: too small and it stutters, too large and it feels remote.
4. **Extrapolation (dead reckoning) only when the buffer runs dry**, and
   always capped — a few hundred milliseconds at most. Uncapped extrapolation
   turns one lost packet into a character sprinting through a wall, then
   snapping back.

## 6 · Lag compensation — hit registration that matches what the player saw

The client's world and the server's world are never the same world. If the
server tests a hit against *now*, a shot the player clearly saw land will miss.

- Keep a **short history of every entity's transform** on the server, at least
  `max_RTT + max_interpolation_delay` long.
- On a shot, rewind the world to what the shooter actually saw — their input
  timestamp, or `receive_time − RTT/2 − their interpolation delay` — and run
  hit detection *there*. What counts is when the event happened, not when the
  packet arrived.
- **Clamp the rewind window.** Unbounded rewind is a cheat surface: a client
  that inflates its own latency gets to shoot into the past.
- **Own the trade-off out loud.** You are buying "my shots land" for the
  shooter by selling "I got shot after reaching cover" to the target. Both
  reports are true. Decide which one your game can live with, cap the rewind
  so the second stays rare, and write the decision in `docs/netcode.md` so
  design stops re-litigating it.

## 7 · Deterministic lockstep and rollback — the relay path

Only **inputs** cross the wire (bit-packed, tens of bytes per tick), so
bandwidth is flat no matter how big the world gets. That is why this fits
mobile PvP and why the server can be dumb and cheap.

- **Lockstep** advances a tick only when every player's input for it has
  arrived — the worst connection sets the pace for everyone.
- **Rollback** removes that wait: predict the missing input, keep simulating,
  and when the real input arrives, restore the frame it belongs to and
  re-simulate forward. It does not remove latency; it removes *waiting*.
  [GGPO](https://github.com/pond3r/ggpo) is the open-source reference
  implementation — read it before writing your own.

**Determinism is the entry fee.** Same start state + same inputs must produce
bit-identical output on every device, or two players are playing different
games. A `0.000001` divergence compounds through physics until the worlds
have nothing to do with each other.

- Fixed-point math (or a deterministic soft-float) and your own physics —
  mainstream engine physics is not deterministic across devices, and neither
  is the platform `float`. Costly, and the cost is the decision in §0.4.
- One seeded PRNG, advanced only inside the simulation.
- Fixed, data-independent update order — never iterate a hash map.
- No wall clock, no frame delta, no render-side input inside the sim.

**Rollback machinery:**

- **State history**: serialize every sim frame into a fixed-size ring
  (30 frames at 30 Hz ≈ 1 s of headroom is a sane start). If a frame cannot
  be serialized in well under a tick, rollback is not available to you.
- **Input delay** — a small fixed local delay buys smoothness. Make it
  *dynamic* against measured RTT; a static high delay makes the character
  feel like a truck.
- **Prediction frames** — how far ahead you will guess. Make this dynamic
  against *device performance*, and scale it **down** on weak hardware: a
  rollback of N frames runs N ticks inside one frame, and on a low-end phone
  that hitch is worse than the desync it was hiding.
- **Detect desync loudly.** Hash the sim state every N frames, exchange
  hashes, and on mismatch save both input logs and end the match cleanly. A
  silent desync is two players in different realities, both blaming you.
- The sim must run **allocation-free** on your worst supported device. Pool
  everything; a GC pause inside a rollback is a visible stutter.

## 8 · Assume a hostile internet

Between your two endpoints sit DHCP and DNS, a NAT, a kernel at each end,
and a chain of routers choosing the next hop by ARP inside each network and
by BGP between autonomous systems. You control none of it. The levers you
actually own are three: **what you put in the packet**, **where you place
the endpoint**, and **what you do between receiving and replying**.
Everything below follows from that.

- **The network is heterogeneous.** Every OS, router and carrier has its own
  defaults (TTL, MTU, NAT timeouts). Prefer boring, standard protocols; every
  bespoke behaviour is one more thing that works on your desk and nowhere else.
- **The network is unstable.** Loss and timeouts are the normal case, not the
  exception. Every non-realtime call gets a timeout, bounded retries with
  exponential backoff **plus jitter**, and a circuit breaker.
- **Transfer costs money.** Serialization, encryption and egress are billed
  in CPU and cash. Optimize the layer you actually control.
- **Traffic takes the cheapest route, not the shortest.** Your İzmir players
  can reach your İstanbul server via Ankara and nobody will tell you. So:
  measure RTT continuously **per region and per ISP** from real clients, alert
  on shifts in the distribution rather than the mean, and keep a synthetic
  traceroute probe per major ISP — a route change explains most unexplained
  latency step-changes.
- **Know your physical floor.** Light in single-mode fibre travels ~1 km per
  5 µs, so the one-way propagation floor is `path_km × 5 µs` and the RTT floor
  is `path_km × 10 µs`. Real fibre runs roughly 1.3–1.5× the great-circle
  distance. İstanbul↔İzmir (~330 km straight, ~450 km of fibre) floors around
  4–5 ms; a measured 20 ms means ~15 ms of equipment, queuing and your own
  code. That gap — not the raw RTT — is what you compare between two ISPs or
  two colocations, and it is the only part you can fix.

## 9 · Verification — netcode you cannot reproduce is netcode you cannot ship

- **Never develop on a LAN.** Force latency, jitter and loss into every dev
  build (Network Link Conditioner, `tc netem`, or a fake in-process
  transport). Make ~150 ms / 5% loss the *default* dev profile so problems
  appear while you are writing them.
- **Deterministic sim → record inputs, replay offline, assert the same state
  hash.** That is the entire test suite for lockstep/rollback, and it is cheap.
- **Keep a desync corpus.** Every mismatch report saves both input logs; each
  fixed bug becomes a regression test.
- **Soak.** Run headless clients — one full match, then ten — against a real
  server for hours, watching CPU *per match* and egress *per match*. Those two
  numbers are your price.
- **Measure at the player, not at the server.** p50/p95/p99 RTT, jitter, loss,
  reconnects, UDP-fallback rate, desync rate, rollback depth — sliced by
  region, ISP and device tier. A good mean hides the players who are leaving.

## 10 · The trap: you choose where to pay, not whether

Two shipped real-time mobile games, opposite choices, same lesson:

- **Server-authoritative physics** (*Headball Online 2*): server owns physics,
  score, timing and goals; the client predicts locally for feel. Matches stay
  in sync and cheating is hard — but the simulation cost lands on your fleet,
  so server CPU (and the bill) grows with the player count.
- **Client-deterministic relay** (*Goalbattle*): the server never runs physics,
  only relays inputs and watches for desync — servers get cheap and dumb. The
  bill moves to engineering: a deterministic math and physics layer, and
  making it run allocation-free on the weakest phone you support. Shipping
  that before it is right costs ranked-play retention, which is far more
  expensive than servers.

Write which one you chose in `docs/netcode.md`, with the number that justified
it. Then hold the line — the mechanics you can afford are downstream of this
choice, so it has to be made before design finalizes anything.

**Other traps worth naming:** UDP-only with no fallback · judging netcode by
mean latency · a well-known port number · unbounded extrapolation or rewind ·
a rollback budget tuned on a flagship phone · letting design promise a
mechanic the topology cannot pay for.
