# Thread analysis: [PATCH v2 0-7/7] seg6: add SRv6 Mobile User Plane (RFC 9433) behaviors

Working document for the Netdev 0x1A "New Age Tooling BoF" talk
*"The Cost Asymmetry of AI-Generated Code: a Case Study in the SRv6 Subsystem"*
(Andrea Mayer, Stefano Salsano).

Source: full mailing-list thread (28 messages, May 5 - Jun 23, 2026) in
`PATCH-v2-1-7-seg6-add-End.MAP-behavior.mbox/`.

---

## 1. The patchset at a glance

| | |
|---|---|
| Submitter | Yuya Kusakabe |
| Content | 6 SRv6 Mobile User Plane behaviors (RFC 9433): End.MAP, End.M.GTP4.E, End.M.GTP6.E, End.M.GTP6.D, End.M.GTP6.D.Di, H.M.GTP4.D + documentation |
| Size | **5,155 insertions, 207 deletions, 12 files** |
| Kernel code | ~2.2k lines added to `net/ipv6/seg6_local.c` (file grows from ~2.7k to ~5k lines — reviewer's figures) |
| Selftests | 2,402 lines of shell (6 scripts), **47% of the series** |
| v1 → v2 turnaround | v1 posted May 4, v2 posted May 5 (**~1 day**), v2 fixes driven by netdev CI |
| Companion series | iproute2-next series posted the same day |

## 2. Timeline (the asymmetry, measured)

| Date (2026) | Event |
|---|---|
| May 4 | v1 posted |
| May 5 | **v2 posted (~1 day later)**: 5,155 LoC. Jakub Kicinski asks to downgrade to RFC until review tags arrive |
| May 8 | Andrea announces he will review the whole series (kernel + iproute2) |
| May 10 | Submitter self-reports that patch 4 implements the wrong RFC section |
| May 16 | Review of cover letter (design-level issues) |
| May 19 | Review of patch 1/7 (End.MAP) |
| May 27 | Review of patch 2/7 (End.M.GTP4.E) |
| Jun 5 | Review of patch 3/7 (End.M.GTP6.E) |
| Jun 7 | Reviews of patches 4/7 and 5/7 (End.M.GTP6.D, D.Di) |
| Jun 8 | Reply on cover-letter thread: rework plan agreed |
| Jun 11-23 | Submitter replies accepting essentially every point |

Bottom line:

- **Generation cost**: ~1-2 days per iteration for a 5.2k-line series.
- **Review cost**: **one month** of expert review (8 emails, ~2,300 lines of review text) covering only **5 of 7 patches**. Patches 6/7 (H.M.GTP4.D) and 7/7 (docs) were never individually reviewed: the process converged on "start over" first.
- **Outcome**: nothing mergeable. Full rework agreed: new lwtunnel encap type, new `SEG6_MOBILE_*` attribute namespace, one series per behavior.
- **Side effect**: the review *increased* the maintainer's workload — Andrea now leads two prerequisite workstreams (SRv6 drop-reason prep series; fix for the pre-existing NF_HOOK dst/cb issue) that the submission surfaced.

## 3. Bug and issue catalog

Roughly **40 distinct findings** across 8 review emails. Grouped by severity class (most talk-worthy first).

### 3.1 Specification errors (code that implements the wrong thing, confidently)

- **Patch 4 (End.M.GTP6.D) implements neither RFC 9433 §6.3 nor §6.4** — it writes orig-DA into SRH[0] *and* Args.Mob.Session into SRH[1], a hybrid matching no section. Comments cite section/step numbers precisely (§6.3 S08, §6.5 S01) while the code does something else. First noticed by the submitter himself 5 days after posting.
- **RFC 6040 mis-cited** in patches 2 and 3 for DSCP/Hop-Limit propagation on a protocol *conversion*; RFC 6040 scopes ECN in IP-in-IP tunnels. Submitter: "the citation was over-scoped."
- Comments describing non-existent code: End.MAP comment says "decrement the Hop Limit" but the code doesn't (v1 did — and double-decremented; v2 removed the code, kept the comment). Another comment claims the packet "traverses NF_INET_LOCAL_OUT" when `dst_output()` only traverses POST_ROUTING.
- Commit message of patch 1 lists a drop reason (`SEG6_MOBILE_HOP_LIMIT_EXCEEDED`) the code never adds — flagged by **Sashiko, the Patchwork AI reviewer**.

### 3.2 Security-relevant validation bugs

- **Malformed SRH accepted as if absent, HMAC silently bypassed.** `seg6_mobile_get_validated_srh()` cannot distinguish "no SRH" from "malformed/truncated SRH" (`seg6_get_srh()` returns NULL for both), so every behavior that tolerates SRH-less input silently processes malformed SRH packets and never reaches HMAC validation. A validation helper that *looks* like defense and structurally cannot do its job.
- **HMAC invalid by construction** on the D-side behaviors: per-packet fields (Args.Mob.Session, preserved DA, saddr/daddr) are stamped *after* `seg6_do_srh_encap()` has computed the HMAC, so a configured HMAC never verifies. Likewise `skb->csum` goes stale (fields written after `skb_postpush_rcsum()`).
- **Fragmented outer packets parsed as complete**: `frag_off` never checked after `ipv6_skip_exthdr()`, in every behavior.

### 3.3 The selftest that passed by coincidence (the killer anecdote)

Sashiko flagged that End.MAP's DA rewrite breaks the ICMPv6 checksum (pseudo-header includes the DA). **The selftest passed anyway.** Andrea dug in:

> "2001:db8:f::1 and 2001:db8:2::e have the same 16-bit word sum
> (0x000f+0x0001 = 0x0002+0x000e), so the checksum stays valid by
> coincidence. Changing nh6 to 2001:db8:2::2 makes the ping fail
> with Icmp6InCsumErrors."

2,402 lines of selftests (47% of the series), green in CI — and the one real
data-path bug in patch 1 was masked by the test's own address plan. Green
tests generated alongside the code they test measure consistency, not
correctness.

### 3.4 UAPI design flaws (the irreversible category)

Andrea's refrain: *"UAPI cannot be undone once merged."*

- **Required attribute with no effect**: in End.M.GTP4.E with `v4_mask_len=32` (the documented example!), the mandatory `src` attribute is read and never used.
- **Attribute that consumes 8 of 128 bits**: with `v4_mask_len=24`, three wildly different `src` values produce the same IPv4 SA; 12 of 16 bytes accepted and discarded.
- **Zero-filled IPv4 DAs** that look like network addresses (`X.Y.Z.0`) for `v4_mask_len<32` — a case RFC 9433 doesn't define. Submitter's answer: no practical use, drop the attribute entirely.
- **Semantic overload of existing UAPI**: `NH6` (next-hop → replacement SID / prefix template), `SRH` (verbatim → per-packet augmented), `OIF` (output device → VRF selector), `src` (verbatim IPv6 SA in some behaviors, bit template in others).
- **Unconstrained values**: PDU Type accepts 0-15 when only 0/1 are meaningful; can't be tightened post-merge.
- **Architectural misplacement**: H.M.GTP4.D is not an endpoint behavior (not bound to a SID, receives IPv4), yet was shoehorned into seg6_local by relaxing the ETH_P_IPV6 guard.

### 3.5 Data-path correctness and performance

- Missing `iptunnel_handle_offloads()` in every encap behavior → GSO broken.
- Per-packet FIB read (rcu + `container_of`) for a value known at configure time.
- End.M.GTP4.E is the only seg6_local behavior using the *output* path, silently bypassing NF_INET_FORWARD (nftables/iptables rules don't see the traffic).
- Open-coded UDP6 checksum setup instead of `udp6_set_csum()` (which also handles GSO).
- Duplicate PDU Session Container silently overwrites QFI.
- `input_action_end()` fallback paths that can only ever drop ("correct per RFC" but functionally dead).

### 3.6 Structure and style

- **Five near-identical copies** of the same GTP-U parse / strip / push logic — and the *small differences between copies are where the bugs hide* (HMAC validated in some behaviors, not others; different drop reasons for the same failure).
- Dead code: defensive checks that cannot trigger (and would fail *silently* if they did, emitting SA 0.0.0.0 packets), dead initializers, `(void)` casts on unused variables.
- Drop reasons misused as a category: duplicates of existing generic reasons (NOMEM, MTU_EXCEEDED), `INVALID_SRH_SL` as a catch-all including HMAC failure, `BAD_INNER` for outer-header failures.
- Style drift from the surrounding file (reverse Christmas tree, anonymous `{}` scope blocks, redundant casts) — noticeable precisely because the file has a consistent style.

## 4. Patterns and anti-patterns of AI-generated kernel code (thesis material)

1. **Plausible-but-wrong**: the code is idiomatic, comments cite RFC sections and step numbers with false precision, commit messages are textbook-perfect — and the behavior implements the wrong section of the spec. Surface signals of quality are decoupled from correctness.
2. **Validation theater**: helpers named and shaped like validation that structurally cannot distinguish the cases they claim to handle; defense-in-depth checks that cannot trigger and fail silently.
3. **Tests that measure consistency, not correctness**: generated tests share the generator's blind spots (only happy paths; address plans that mask checksum bugs). 47% of the series was tests, CI was green, and the bugs were all still there.
4. **UAPI without a designer**: attribute semantics assembled by pattern-matching on existing code (NH6, SRH, OIF reuse) rather than by design; required-but-unused parameters; unconstrained ranges. This is the most dangerous category because **UAPI mistakes are forever**.
5. **Copy-paste amplification**: one flawed template instantiated five times; review effort multiplies while generation effort doesn't.
6. **Volume as a denial-of-service on review**: 5.2k lines, one day per iteration. The cheap-to-generate/expensive-to-review ratio is the structural problem — everything above is a symptom.
7. **The cooperative loop**: every review point is accepted instantly, eloquently, with perfect summaries ("Will do exactly as you describe"). Zero pushback in ~40 findings. Pleasant — but a reviewer learns nothing about which parts the submitter actually understands.

## 5. AI in the loop, on both sides

- **AI as reviewer: marginal net contribution, and only under human verification.** Sashiko (the AI review system) produced 22 findings across all 7 patches, but source-level verification (§6) leaves a harsh ledger: it reproduced only ~9 of the human review's ~40 findings (§6.3.1); of its 5 unique findings on the common perimeter, 2 are real but minor and 3 are false positives built on invented kernel mechanisms — including its single Critical (§6.5); and it found zero findings in the three classes that determined the thread's outcome (spec conformance, UAPI design, architecture). Its genuine value in this thread reduces to two flags Andrea credited in-thread — the commit-message/code mismatch and the ICMPv6 checksum break — and even the better of those only became actionable because Andrea reproduced it, explained *why* the green selftest masked it, and derived the fix ("The AI bot was right, but when I ran the selftest it passed"). Every AI finding, right or wrong, consumed expert verification time: on this series, verification debunked more than it confirmed.
- **AI provenance was never disclosed in-thread.** No message states the patchset was AI-generated. Observable signals: the 1-day v1→v2 turnaround on a 5.2k-line series; the wrong-section bug (hard to explain if the code had been written by reading §6.3); uniform structure across behaviors; late replies carry message-IDs from a single batch (`20260612032313.*`) sent across Jun 19-23, suggesting pre-drafted responses. *For the talk: be careful to present these as signals, not proof — and note that the absence of disclosure norms is itself a finding (cf. README disclaimer: no one can be blamed for getting it wrong).*

## 6. Human review vs AI review, side by side

Source: Sashiko public dashboard for this patchset —
<https://netdev-ai.bots.linux.dev/sashiko/#/patchset/20260505-seg6-mobile-v2-0-9e8022bdfdb6%40gmail.com>.
Model: `bedrock/global.anthropic.claude-opus-4-7`, Sashiko ver. `85e4c207`. All 7 patches reviewed.

### 6.1 The two reviews in numbers

*The findings comparison below is restricted to the common perimeter — cover letter + P1-P5, the patches Andrea actually reviewed. Sashiko also reviewed P6 and P7 (4 more findings), but with no human counterpart there is nothing to compare those against.*

| | Human (Andrea) | AI (Sashiko), same perimeter (P1-P5) |
|---|---|---|
| Elapsed time | ~5 weeks (May 8 – Jun 8) | hours after posting |
| Cost | weeks of maintainer time | ~$34 in tokens on P1-P5 (~$42 for all 7; per-patch $1.12–$9.36) |
| Findings | ~40 | 18 (1 Critical, 5 High, 7 Medium, 5 Low) — the sole Critical is a false positive, see §6.5 |

Sashiko per patch (Critical/High/Medium/Low, "% unique" per dashboard): P1 End.MAP 0/1/3/0 (25%) · P2 GTP4.E **1**/1/2/3 (86%) · P3 GTP6.E 0/1/2/1 (25%) · P4 GTP6.D **0/0/0/1** · P5 D.Di 0/2/0/0. Outside the comparison perimeter: P6 H.M.GTP4.D 0/1/2/0 · P7 docs 0/0/0/1.

### 6.2 What the AI found that the human missed (within P1-P5)

- `__ip_select_ident(net, iph, 1)` under-reserves IP IDs for GSO frames → potential ID collisions (P2). **Verified on kernel source: real bug, low severity in this path.** The mechanism is correct — `inet_gso_segment()` assigns `id++` per segment, so reserving 1 ID for an N-segment frame puts N-1 unreserved IDs on the wire, and the tree idiom is indeed `__ip_select_ident(net, iph, skb_shinfo(skb)->gso_segs ?: 1)` (`iptunnel_xmit()`, which uses it even with DF set). Two context mitigations Sashiko does not mention: (a) the patch hard-codes `IP_DF`, so every emitted datagram is *atomic* and RFC 6864 exempts it from ID-uniqueness (cf. `ip_select_ident_segs()`, which uses `id = 0` in the DF branch) — collisions have no on-wire effect short of DF-clearing middleboxes; (b) GSO on this path is broken anyway until the missing `iptunnel_handle_offloads()` (Andrea's finding) lands — **the two findings are chained: the human's fix is what would make the AI's bug live**. Worth the one-line fix regardless.
  - *Reviewer economics note*: Andrea never audited this detail — and rightly so. Having established that GSO on the path was broken at the root, he provided the upstream fix guidance (exact call placement between `skb_cow_head()` and `seg6_mobile_push_gtpu()`, the two inner-header snapshot prerequisites, `skb_set_inner_protocol()` after the call, "the fix needs end-to-end testing") and moved on. Auditing the IP-ID allocation of a non-functional GSO path would have been premature: a human reviewer triages by priority; the AI audits everything at equal depth. Complementary again — but only one of the two knows what to look at *next*.
- End.M.GTP6.E's GSO/MTU check reads `dst_mtu(skb_dst(skb))` while `skb_dst` is still the ingress SRv6 route that matched the SID, and over-counts overhead on the short-header GTP-U path (P3). **Verified on kernel source: valid finding, minor severity; the comment/code mismatch is its solid core.** The mechanism is correct — in a seg6_local input handler `skb_dst` is the rt6_info set by `ip6_route_input()` (`lwtunnel_input()` dispatches on `skb_dst(skb)->lwtstate`), the egress dst is only set later inside `seg6_lookup_any_nexthop()`, and `dst_mtu()` → `ip6_dst_mtu_maybe_forward()` reads the ingress route's `RTAX_MTU`/device `mtu6` — not the "egress IPv6/UDP/GTP-U path" the comment claims. But it is not structural: an over-MTU GSO packet that slips through is still caught on the real egress by `ip6_forward()`'s `ip6_pkt_too_big()`, with ICMPv6 PTB; the residual harm is false drops — silent, no PTB — when the ingress route under-states the egress MTU, plus 8 bytes of lost MSS margin on the short-header path (`ovhd` always counts `gtp1_header_long` + PDU Session ext, 16 B, vs the 8-B `gtp1_header` actually pushed when `pdu_type_set` is false). Nor is the check "consistent with the tree": upstream seg6_iptunnel.c/seg6_local.c do no pre-lookup GSO/MTU check at all (seg6_iptunnel relies on `iptunnel_handle_offloads()`). Telling detail: the twin check in P2 documents itself honestly as "a conservative bound (the outbound IPv4 route is not known until ip_route_output_key() below)" — only P3's comment claims the egress path.
  - *Reviewer economics note*: same chain as the IP-ID entry above — with `iptunnel_handle_offloads()` missing (Andrea's finding), GSO on this path is broken wholesale, and the offload rework he prescribes would likely restructure or remove this ad-hoc check anyway; the human flagged the root cause invalidating the block, the AI audited its subcases. Whether Andrea saw and triaged the MTU-dst point or overlooked it is not decidable from the thread: his "Same ovhd scoping point as patch 2" comment sits on these very lines, but addresses style, not semantics. The comment/code mismatch survives the rework either way.
- TTL/hop-limit not clamped: emitted outers with ttl 0/1; hop_limit 0 packets reaching local sockets via `ip6_input()`, which skips the expiry check (P1, P2). **Verified on kernel source: false positive on both legs; the supporting kernel claim is fabricated.** *Leg A (P1, End.MAP)*: the "local sockets" scenario cannot occur — End.MAP calls `seg6_lookup_nexthop()`, i.e. `seg6_lookup_any_nexthop()` with `local_delivery=false`, which explicitly discards local-delivery dsts (`IFF_LOOPBACK` filter → blackhole); and the premise proves too much anyway: `ip6_input()` contains no hop-limit expiry check *by design* (RFC 8200: the hop limit is a forwarding budget — a plain hl=0/1 packet addressed to the node is delivered identically with no SRv6 involved). The loop sub-question also fails: every off-box traversal of an End.MAP node charges exactly 1 via `ip6_forward()` (`hop_limit <= 1` → ICMPv6 Time Exceeded + drop, then `hdr->hop_limit--`) — the very reason v2 dropped the explicit decrement (cover letter: v1 double-decremented, hlim 64→62; v2 verified 64→63). RFC 9433 §6.2 S01-S04 does place check+decrement in the endpoint pseudocode, but the kernel realizes the same per-node cost in the forwarding path — an equivalence upstream already embraces: zero `hop_limit` references in all of seg6_local.c, End/End.X included. *Leg B (P2)*: the claim "iptunnel_xmit and similar tunnel encap paths enforce a minimum TTL on the new outer" is false at every checkable point — `iptunnel_xmit()` assigns `iph->ttl = ttl` verbatim; the `if (ttl == 0)` branches in `ip_tunnel_xmit()`/`ip_md_tunnel_xmit()` are an inherit-default for *unconfigured* TTL (inner ttl 1 → outer ttl 1); geneve's `ttl = ttl ?: ip4_dst_hoplimit()` likewise; vxlan's only special case *sets* ttl = 1 (multicast default) — the very value the finding says encap paths avoid. Copying hop_limit→ttl verbatim is exactly what tree encaps do under ttl-inherit and is correct TTL semantics for a converted packet whose budget is spent (an upward clamp would enable loops; RFC 9433 §6.6 says only "set the ... Hop Limit field", no minimum); an emitted ttl=0 outer requires an on-link sender that transmitted hl=0 in the first place — anomalous input, dead packet at the first conformant router (RFC 1812). Third fabricated-mechanism instance, after the §6.5 UAF and the `.static_headroom` route-headroom claim above.
  - *Reviewer economics note*: Andrea had the correct model all along and spent one sentence on it — his P1 nit: the comment "says 'decrement the Hop Limit' but the code does not do it explicitly. The forwarding path handles it (ip6_forward)." That sentence is, unknowingly, the rebuttal of both of Sashiko's questions. On P2 he touched hop limit only through the RFC 6040 mis-citation and never asked for a clamp — consistent with the tree practice he evidently knew. The human's one-line comment fix outperforms the AI's two speculative attack scenarios built on a nonexistent kernel convention.
- Missing `.static_headroom` → per-packet COW on the data path (P2). **Verified on kernel source: largely a false positive — the claimed mechanism does not exist; residual value is a consistency/future-proofing nit.** Three verified facts: (1) the only consumer of `lwtstate->headroom` we could locate is `lwtunnel_headroom()` (`include/net/lwtunnel.h`), which feeds *route MTU accounting* (`ip_dst_mtu_maybe_forward()` / `ip6_mtu()`), not skb allocation — nothing in the kernel pre-reserves skb headroom from it, so "the route pre-reserves headroom for the outer" is fabricated; (2) `lwtunnel_headroom()` returns 0 unless the state is output/xmit-redirect, and seg6_local builds every state as `LWTUNNEL_STATE_INPUT_REDIRECT` (`seg6_local.c`), so the value is inert for *all* seg6_local behaviors — including End.B6.Encap, whose `.static_headroom = sizeof(struct ipv6hdr)` Sashiko cites as the proof-by-sibling; (3) "an allocation/copy per packet" is unsupported — `skb_cow_head()` reallocates only when actual headroom falls short or the header is cloned, received skbs typically carry ≥ `NET_SKB_PAD`, and the finish path already reserves the worst case once. Declaring the encap overhead stays reasonable as *future-proofing* for the planned new encap type (output/xmit semantics would make the MTU accounting live) — but not for the stated reason.
  - *Human-vs-AI vignette*: Andrea reviewed the very same `skb_cow_head()` lines and drew the correct, modest conclusion — "The caller already reserves worst-case headroom, so the skb_cow_head() calls inside seg6_mobile_push_gtpu() are always no-ops. Could they be removed?" — while the AI, from the same lines, inferred a kernel mechanism that does not exist. Same evidence, opposite epistemics: the human under-claims, the AI over-explains. Together with the §6.5 UAF and the TTL-clamp convention (entry above), the pattern is now established across three findings: Sashiko's weak spot is *mechanistic claims about kernel plumbing* stated with idiomatic confidence.

### 6.3 What the human found that the AI missed

The irreversible and the intentional:

- **The deepest bug in the series**: patch 4 implements *neither* RFC 9433 §6.3 nor §6.4. Sashiko gave that very patch its cleanest score (0/0/0/1). Checking code against the *intent* of a spec is where it whiffed completely.
- **The entire UAPI-design category**: required-but-unused `src`, `v4_mask_len` semantics, NH6/SRH/OIF semantic overload, unconstrained PDU Type — the "cannot be undone once merged" reasoning.
- **Architecture and process**: H.M.GTP4.D doesn't belong in seg6_local; module split; one series per behavior; drop-reason prep series; duplication across the five behaviors.
- HMAC invalid-by-construction (fields stamped after the SRH is signed) and stale `skb->csum`.
- **The offload-integration bug**: missing `iptunnel_handle_offloads()` breaks GSO of the outer tunnel — found by Andrea in P2 (May 27) with full placement analysis (inner header snapshot mechanics, `skb_set_inner_protocol()`), then referenced in his P3/P4 reviews ("Same missing iptunnel_handle_offloads() as patch 2"). **Absent from Sashiko's 22 findings.** Neat symmetry: both reviews found GSO problems, on entirely different facets, with zero overlap.
- **Empirical verification**: only the human ran the selftest, noticed it passed despite a real bug, and worked out the checksum coincidence.
- Negotiating the rework: deciding what blocks merge vs what's a nit, and steering the submitter to a viable resubmission plan.

### 6.3.1 The reverse ledger: how much of the human review did the AI reproduce?

Counting Andrea's ~40 findings as thematic clusters (his "Same X as patch 2" back-references folded into one cluster each), **Sashiko reproduced ~9 of ~40 (~22%)** — i.e. half of Sashiko's 18 findings on P1-P5 coincide with a quarter of Andrea's:

| Shared finding | Who did it better |
|---|---|
| Drop reason in commit msg but not in code (P1) | AI-first: Andrea credits Sashiko in-thread |
| ICMPv6 checksum break on DA rewrite (P1) | AI flag, but the reproduction and the coincidence analysis are Andrea's (and it is not in the current dashboard run — §6.6 caveat) |
| Malformed SRH treated as absent → HMAC bypass (P1) | Sashiko's framing sharper (`seg6_require_hmac` bypass scenario) |
| Hop-limit comment/code mismatch (P1) | Andrea: one correct sentence vs two invalid AI attack scenarios (§6.2) |
| INVALID_SRH_SL granularity/naming | Partial: Andrea also covers NOMEM/MTU duplication, BAD_INNER misuse, and proposes the prep series — Sashiko only the naming |
| cb/dst corruption across NF_HOOK | Andrea: pre-existing root cause identified (7a3f5b0de364) and owned as a fix workstream; Sashiko adds concrete clobber chains |
| Locator length derived from the FIB (P2) | Even: both propose the explicit attribute |
| Fragmented outer not handled | Sashiko deeper on P4 (non-first frags → End path; first frags → truncated inner) |
| RFC 6040 mis-citation | Sashiko more precise ("uniform mode" vs ECN state machine) |

**~31 of Andrea's findings are invisible to Sashiko.** The most critical misses, ranked:

1. **P4 implements neither RFC 9433 §6.3 nor §6.4** — the deepest bug in the series, on the patch Sashiko scored cleanest (0/0/0/1). Checking code against the *intent* of a spec is the number-one hole.
2. **HMAC invalid-by-construction** (per-packet fields stamped after the SRH is signed) **+ stale `skb->csum`** — a security feature silently 100% broken; zero AI signal.
3. **The entire UAPI-forever block**: required-but-unused `src` (in the documented example!), the 8-of-128-bits template, DA zero-fill, NH6/SRH/OIF semantic overload, unconstrained PDU Type. Irreversible after merge — the longest-lasting damage class, entirely absent from Sashiko's 22 findings (its only UAPI-adjacent item is the drop-reason naming).
4. **NF_INET_FORWARD bypass** via `dst_output()` (the only behavior on the output path): firewall rules silently never see the traffic, and the comment claims LOCAL_OUT. Operationally and security-relevant.
5. **Missing `iptunnel_handle_offloads()`** — the root cause of the GSO breakage. Peak irony: Sashiko produced three GSO-adjacent findings (IP-ID, MTU, TTL-clamp) while missing the root cause that dominates all of them.
6. **Architecture and process**: H.M.GTP4.D misplaced in seg6_local, module split, per-behavior series, ×5 duplication — the findings that actually determined the thread's outcome ("start over").
7. Minor but real: duplicate PDU Session Container silently overwriting QFI; selftests with scapy dependency and no adversarial cases; the dead defensive check emitting SA 0.0.0.0.

The structural reading mirrors the false-positive pattern (§6.5): Sashiko excels *and* fails **inside the mechanics of a single hunk**; Andrea dominates everything that requires **context beyond the patch** — the RFC read end-to-end, coherence between patches 4 and 5, UAPI semantics across existing behaviors, empirical execution, the merge decision. Of the three finding classes that determined the outcome (spec, UAPI, architecture), Sashiko found zero.

### 6.4 Where they overlapped

Absent-vs-malformed SRH → HMAC bypass (Sashiko's framing arguably sharper: an explicit `seg6_require_hmac`-bypass attack scenario); hop-limit comment/code mismatch; drop-reason naming; RFC 6040 mis-citation (Sashiko more precise: verbatim copy = "uniform mode", not the RFC 6040 ECN state machine); fragility of deriving locator length from the FIB (both proposed the explicit attribute); fragmented-outer handling (Sashiko deeper on P4: non-first fragments silently take the End path, first fragments emit truncated inner payloads); **the `skb->cb`/dst corruption across NF_HOOK** — Andrea flagged it in the cover letter (IPCB/IP6CB aliasing, NULL or unrelated dst in the finish callback, traced to the pre-existing 7a3f5b0de364 and taken on as his own fix workstream); Sashiko added concrete clobber call-chains (defrag → `IPCB` memset, flowtable, ovs) and attacker-influence framing on P2/P5.

### 6.5 The AI's false positives (verified on kernel source)

Source-level verification of the five AI-only findings leaves three false positives, all sharing the same signature: **a nonexistent kernel mechanism asserted with idiomatic confidence**. Two are documented inline in §6.2 (the `.static_headroom` "route pre-reserves headroom" claim; the "encap paths enforce a minimum TTL" convention). The third — and the most expensive, being the AI's single Critical — is detailed here.

Sashiko's single Critical finding — *"`slwt->oif` read after `skb_dst_drop()`: if the skb held the last dst reference, `dst_destroy()` runs, `lwtstate_put()` frees `slwt`, use-after-free"* — does not hold up against the actual kernel code:

1. `skb_dst_drop()` → `refdst_drop()` → `dst_release()` only when the dst is refcounted (`include/net/dst.h`). Input-path dsts are typically **noref** (`skb_dst_set_noref`, valid only under RCU — cf. the `WARN_ON(!rcu_read_lock_held())` in `skb_dst_force()`), in which case `skb_dst_drop()` releases nothing at all.
2. Even for a refcounted dst, `dst_release()` never destroys synchronously: on last reference it does `call_rcu_hurry(&dst->rcu_head, dst_destroy_rcu)` (`net/core/dst.c`). The `lwtstate_put()` that would free `slwt` sits inside `dst_destroy()` — always **deferred past an RCU grace period**. (The synchronous variant, `dst_release_immediate()`, is not used by `skb_dst_drop()`.)
3. The seg6_local input handler executes in the packet receive softirq inside the RCU read-side section held by `__netif_receive_skb_core` (and on the nf_queue reinject path, `nf_queue_entry_get_refs()` takes a real dst reference via `skb_dst_force()`). A grace period cannot elapse while that section is open, so `slwt` memory is guaranteed valid for the whole function — including after `skb_dst_drop()`.

The claim is plausible-sounding lifetime reasoning that ignores RCU deferral. At most it is a style remark (snapshotting `slwt` fields before the drop, as the End.DX4/DX6 handlers happen to do, is the cleaner pattern). *This matters doubly for the talk: (a) the AI's only Critical was noise, and refuting it took exactly the expert knowledge AI review is meant to economize; (b) false positives at Critical severity are the expensive kind — a maintainer cannot ignore them. Across the three false positives the failure mode is uniform (invented plumbing: RCU-deferred lifetime, route headroom, TTL-clamp convention), which mirrors Sashiko's own "plausible-but-wrong" pattern from §4 — AI review exhibits the same failure mode as AI code.*

### 6.6 Caveats and the thesis

- Sashiko is probabilistic — its own docs claim 53.6% of historical `Fixes:`-tagged bugs found, varying run to run. The P4 near-miss is exactly that variance.
- The per-patch severity counts match the findings visible in the dashboard text one-for-one (checked for all 7 patches), so the `[ ... ]` elisions hide only diff context, not findings — claims about what Sashiko did *not* flag in this run are on solid ground.
- Andrea credits Sashiko in-thread for the ICMPv6 checksum flag; it is not among the four findings of the current dashboard run for P1 (a different run, per the probabilistic caveat above) — cite carefully.
- The dashboard's "% unique" (25% on P1, 86% on P2) suggests Sashiko itself tracks overlap with the list discussion.

**Thesis for the talk — AI review does not fix the cost asymmetry; it re-enacts it.** The talk's core claim is that AI-generated code is cheap to produce and expensive to review. The obvious rebuttal — "then let AI do the reviewing" — is what this comparison tests, and the verified ledger answers it. On the same perimeter (P1-P5): every finding class that determined the thread's outcome (spec conformance, UAPI-forever, architecture) came from the human and was invisible to the AI (§6.3.1); the AI reproduced ~22% of the human review; its unique contribution nets out to **two real but minor data-path bugs** (IP-ID under-reservation, itself gated on the human's offload fix; the P3 ingress-MTU/comment mismatch) against **three false positives, each resting on a kernel mechanism that does not exist** (the §6.5 UAF ignoring RCU deferral; the `.static_headroom` "route pre-reserves headroom" claim; the "encap paths enforce a minimum TTL" convention) — including its single Critical, the kind a maintainer cannot ignore. The failure signature is the same *plausible-but-wrong* pattern as AI code (§4): mechanistic claims about kernel plumbing stated with idiomatic confidence. So the asymmetry reappears one level up, intact: AI findings are cheap to generate and expensive to verify — on this series, expert verification *debunked more than it confirmed* — and the scarce, non-substitutable resource remains what it was before: expert human attention, which AI review consumes rather than replaces.

## 7. From verdict to proposals: what should change

The harsh ledger of §6 is not an argument for discarding AI review — it is a measurement of how far it currently is from useful, and a map of where the improvement margins are. Two concrete proposals for the BoF.

### 7.1 AI review must be improved — and the margins are wide

- **Feedback loops to kill false positives.** All three §6.5-class false positives were *mechanically debunkable* from kernel source — which means they are also mechanically preventable. Today the maintainer's verdict on a finding (confirmed / debunked) goes nowhere: the dashboard tracks "% unique" but has no channel for "% wrong". Close the loop: a maintainer marking a finding as a false positive should become evaluation and tuning signal for the reviewer. Every debunk of the kind documented in §6.2/§6.5 is, potentially, a regression test for the review agent — this very case study is such a dataset.
- **Domain-specific knowledge, owned by maintainers.** The three false positives share one signature: *generic* plumbing claims applied to a subsystem whose actual rules the model didn't know (RCU-deferred dst/lwtstate lifetime; `lwtunnel_headroom()` semantics for input-redirect states; TTL-inherit conventions in encap paths). That is precisely what per-subsystem knowledge fixes. The mechanism already exists: Sashiko runs on an open-source set of per-subsystem and generic prompts (Chris Mason's review-prompts repo, cited on its About page). What is missing is the contribution pipeline: **maintainers must/can contribute the domain-specific review prompts for their own subsystem** — effectively MAINTAINERS-owned review policy, versioned and reviewed like code. The SRv6 rules that would have prevented all three false positives fit in a page.

### 7.2 A desk-rejection threshold for submissions

Journals have **desk rejection**: the editor rejects a manuscript without full peer review when quality is evidently below the bar. Kernel review has no formalized equivalent — the closest things in this thread were netdev CI and Jakub's RFC-downgrade, both far short of a rejection gate. The README of this case study already states the instinct: "an average maintainer would normally have rejected this patchset after the first 5 to 10 bugs." The proposal is to formalize it: **below a threshold of evident quality, the series is rejected wholesale, with the burden of proof shifted back to the submitter** (composable with the verified-submission gates of Rajat's talk in this same BoF).

The strong point: in this case, *trivial* indicators would have inferred — with probability close to 1 — that the code was AI-generated with low-quality prompting, before any deep review started:

- a 5,155-line series covering **six behaviors at once**, with v1→v2 turnaround of one day;
- a *required* attribute that is unused **in the patchset's own documented example** (`src` with `v4_mask_len 32`) — something no author who ran their own example can produce;
- comments describing code that does not exist (the hop-limit decrement, the LOCAL_OUT claim) and citations off by one RFC section (§6.3 vs §6.4) or out of scope (RFC 6040);
- selftests that are 47% of the series and all green, yet arranged so as to miss every real bug (checksum-neutral address plans);
- five near-identical copies of the same logic whose small divergences are exactly where the bugs sit;
- (meta, softer) batch message-IDs in the submitter's replies.

None of these indicators requires finding a single bug — they are cheap, mechanical signals whose job is to *price the review before it starts*. Framed for the BoF: a desk-rejection threshold is not anti-AI; it is review-economics. It protects the scarce resource (§6.6) that the cost asymmetry attacks: expert maintainer attention. Disclosure norms, desk rejection, and verified-submission gates compose into a pipeline where the default cost of a low-effort 5k-line submission returns to its sender.

## 8. What the process got right (worth saying at a BoF)

- Jakub's early move: downgrade to RFC until review tags arrive — a lightweight guard that CI had already been burned (v2's fixes were "all reported by netdev CI").
- The review converged on *process* fixes, not just code fixes: per-behavior series (End.DT4/DT6/DT46 precedent), separate module + Kconfig, new encap type discussed with the lwtunnel community first, prep series for shared drop reasons.
- The submitter's cooperation, whatever produced it, kept the thread civil and convergent.

## 9. Key quotes (verified against the mbox)

> "Could you switch to posting this as an RFC until you gather some review tags?" — Jakub Kicinski, May 4

> "It is a substantial addition so I want to take the time to review it properly." — Andrea, May 8

> "A required UAPI attribute that has no effect on the output cannot be relaxed after merge, so this needs to be revised." — Andrea, May 27

> "The AI bot was right, but when I ran the selftest it passed. [...] the checksum stays valid by coincidence." — Andrea, May 19

> "As far as I can see, this matches neither Section 6.3 (Args.Mob.Session in SRH[0], no D) nor Section 6.4 (D in SRH[0], no Args.Mob)." — Andrea, Jun 7

> "Producing a 5000 lines-of-code patchset with AI may take 1 or 2 days of work, while manually reviewing it takes a Linux kernel maintainer 5 to 10 times longer." — README / talk proposal

## 10. Suggested skeleton for the ~10-minute slot (8-9 slides)

1. **Title + disclaimer** — "not here to blame anyone" (the README disclaimer, verbatim: it sets the tone).
2. **The patchset** — 6 RFC 9433 behaviors, 5,155 LoC, 12 files, v1→v2 in one day; looks great: docs, selftests (47%), perfect commit messages.
3. **The timeline** — one month of expert review for 5 of 7 patches; outcome: start over. The asymmetry, measured.
4. **Anatomy of the findings** — ~40 findings by severity: spec errors / security validation / UAPI-forever / data path / structure.
5. **The checksum that passed by coincidence** — the End.MAP story (Sashiko flag → green selftest → equal 16-bit word sums). One slide, one anecdote, carries the whole "tests ≠ correctness" point.
6. **Wrong section of the RFC** — comments citing §6.3 step numbers over code implementing neither §6.3 nor §6.4. "Plausible-but-wrong." Punchline: Sashiko gave this patch its cleanest review (0/0/0/1).
7. **Human vs AI review, same patches** — the §6.1 numbers table (common perimeter P1-P5) + the verified ledger of the AI's unique findings: 5 flagged → 2 real but minor, 3 false positives *each built on an invented kernel mechanism* (RCU-deferred UAF, route headroom, TTL-clamp convention — §6.2/§6.5), including its only Critical; debunking them took exactly the kernel expertise AI review is meant to economize. The human (weeks) caught spec-intent, UAPI-forever, architecture, and the offload-integration bug the AI missed — and was the only one who *ran* anything, or could referee the false positives. Even on GSO they found disjoint facets. Punchline candidate: on Andrea's P1 hop-limit nit, one human sentence ("the forwarding path handles it") pre-refuted two AI attack scenarios.
8. **Anti-patterns + lessons** — the 7 patterns of §4 compressed to ~4 bullets; dos & don'ts: disclosure; small series; UAPI designed by hand; don't trust generated tests; treat AI review output as claims to verify, not findings — it fails the same plausible-but-wrong way AI code does; humans keep the merge decision.
9. **Proposals (§7) + discussion hook** — two constructive moves: (a) improve AI review where the margins evidently are: feedback loops on debunked findings ("% wrong" next to "% unique") and maintainer-owned per-subsystem review prompts (the mechanism already exists — review-prompts — the pipeline doesn't); (b) **a desk-rejection threshold**, the journal analogy: this series carried trivial, mechanical, probability-≈1 indicators of low-effort AI generation (required-but-unused attribute in the documented example, comments describing absent code, one-day 5.2k-line turnaround) — reject wholesale below the bar and return the cost to the sender. Hook for the room: who defines the bar, and where does it live (CI? Patchwork? MAINTAINERS)?

---

*All counts and quotes cross-checked against the mbox on 2026-07-11. Message index: patches = msgs 0-7; Jakub = 8, 10; Andrea = 11, 14, 15, 18, 19, 20, 21, 22; Yuya replies = 9, 12, 13, 16, 17, 23-27.*
