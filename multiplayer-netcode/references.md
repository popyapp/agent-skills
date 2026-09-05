# References

This skill is a distillation, not a translation. The engineering judgement,
the topology/protocol framing and the two production post-mortems come from
one source; the wording, structure and the checklists in `SKILL.md` are ours.
Read the originals — they are better than any summary, and they are free.

## Primary source

**"Multiplayer Oyun Geliştirirken Bilmeniz Gerekenler"** (Turkish) by
**Oğuzhan Yılmaz** — Mage Games. Netcode on *Headball Online 2* (Kafa Topu)
and *Goalbattle*.

| Part | Link | What it covers | Maps to |
|---|---|---|---|
| Bölüm 1 (Jun 2023) | [medium.com/mage-games/…bölüm-1](https://medium.com/mage-games/multiplayer-oyun-geli%C5%9Ftirirken-bilmeniz-gerekenler-b%C3%B6l%C3%BCm-1-16113383df3f) | Vocabulary (lag, ping, RTT, jitter, packet loss, reliability); network topologies (P2P, NAT hole punching, client-authoritative, dedicated, relay); TCP vs UDP; port selection | "Reading the symptom", §2, §3 |
| Bölüm 2 (Jul 2023) | [medium.com/mage-games/…bölüm-2](https://medium.com/mage-games/multiplayer-oyun-geli%C5%9Ftirirken-bilmeniz-gerekenler-b%C3%B6l%C3%BCm-2-90cd7327dde1) | The life of a packet end to end (DHCP, DNS, ARP, NAT, BGP/ASN routing); the fallacies to unlearn (the network is heterogeneous, unstable, transfer is not free); cheapest-route-not-nearest; propagation limits | §8 |
| Bölüm 3 (Sep 2026) | [blog.oguzhan.info/…bölüm-3](https://blog.oguzhan.info/multiplayer-oyun-geli%C5%9Ftirirken-bilmeniz-gerekenler-b%C3%B6l%C3%BCm-3-dc6aece9f321) | Frame rate vs tick rate; client-side prediction; server reconciliation; interpolation and the interpolation buffer; extrapolation; lag compensation; rollback; deterministic simulation; lockstep; Headball and Goalbattle post-mortems | §4, §5, §6, §7, §10 |

Author's related piece on making a deterministic simulation survive low-end
Android hardware (zero-allocation, pooling — the cost behind §7):
[Game Developer'lar İçin Mobil Cihazlarda Performans Bilinci](https://blog.oguzhan.info/game-developerlar-i%C3%A7in-mobil-cihazlarda-performans-bilinci-03d56097a2b6).

## What we changed on purpose

- **Propagation math re-derived.** `SKILL.md` §8 states the fibre floor as
  `path_km × 5 µs` one way (`× 10 µs` for RTT), with real fibre running
  ~1.3–1.5× the great-circle distance, and frames the useful quantity as the
  *gap* between measured RTT and that floor. The physical constant is the
  source's (≈0.66 c in fibre, ≈4.9 µs/km); the worked example is ours, so the
  units carry through.
- **Delivery classes over UDP** (§3) and the **verification chapter** (§9) are
  additions — the source names the problems, this skill names the tests.
- **The vocabulary is re-cut as a symptom table.** The source defines the terms;
  here each one is tied to the complaint you will actually receive and to the
  section that fixes it, so an agent can route a bug report.
- **The packet's journey is compressed to its lesson** (§8 opening) rather than
  retold — read Bölüm 2 for the full walk, it is the enjoyable part.
- **`docs/netcode.md`** (§1) is a Popy convention, matching the design-doc-first
  habit in [`game-dev`](../game-dev/).

## Further reading

Carried over from the source's own bibliography, plus the canonical set:

- [What Every Programmer Needs To Know About Game Networking](https://gafferongames.com/post/what_every_programmer_needs_to_know_about_game_networking/) — Glenn Fiedler (Gaffer On Games)
- [Client-Server Game Architecture](https://www.gabrielgambetta.com/client-server-game-architecture.html) — Gabriel Gambetta (the clearest prediction/reconciliation/interpolation walkthrough there is)
- [Source Multiplayer Networking](https://developer.valvesoftware.com/wiki/Source_Multiplayer_Networking) — Valve
- [Latency Compensating Methods in Client/Server In-game Protocol Design](https://developer.valvesoftware.com/wiki/Latency_Compensating_Methods_in_Client/Server_In-game_Protocol_Design_and_Optimization) — Yahn Bernier, Valve (the lag-compensation paper)
- [GGPO](https://github.com/pond3r/ggpo) — open-source rollback networking
- [Minimizing the Pain of Lockstep Multiplayer](https://www.gamedeveloper.com/programming/minimizing-the-pain-of-lockstep-multiplayer) — Game Developer
- [The Poor Man's Netcode](https://etodd.io/2018/02/20/poor-mans-netcode/) — Evan Todd
- [GameNetworkingResources](https://github.com/ThusSpokeNomad/GameNetworkingResources) — curated list
- [Fallacies of distributed computing](https://en.wikipedia.org/wiki/Fallacies_of_distributed_computing) — Peter Deutsch, with the eighth added by James Gosling. §8 is these fallacies applied to games; the source credits them and so do we.
- [Networking zine](https://wizardzines.com/zines/networking/) — Julia Evans, for the layers §8 says you do not control
- [Submarine Cable Map](https://www.submarinecablemap.com/) — why "nearest" and "fastest" are different words

Bölüm 2 closes with two more (Fastly's network-scaling series and a piece on
fixing the internet for real-time applications) — follow them from the post
itself rather than from a link we did not verify.

## Credit

If this skill helps you, the credit belongs to
[Oğuzhan Yılmaz](https://blog.oguzhan.info) — go read the series and follow
the author. Packaged for agents by [Popy](https://popy.app) / Motivolog under
MIT; the underlying articles remain the author's.
