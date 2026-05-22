# Design change: proxy acquisition, OPEN, and layout gating

**Status:** Recorded (v2, revised after the 2026-05-22 protocol-soundness
review). Ready to apply to
`draft-haynes-nfsv4-flexfiles-v2-proxy-server.md`.
**Date:** 2026-05-22
**Scope:** How a client reaches a proxied file; how a PS is bound to a
file; the two filehandles involved and how each is conveyed; how OPEN
and LAYOUTGET are disambiguated and gated while a proxy is active.

This is one structural change; the pieces are interdependent and apply
together. §8 (assimilation) is explicitly deferred to a later change.

## 1. Origin

The review began at one sentence in the Introduction:

> Clients discover that a file is being proxied through the normal
> pNFS layout path -- a layout that names the PS with a new
> data-server flag (FFV2_DS_FLAGS_PROXY) -- and route their I/O
> through it while it is active.

Pulling on it exposed several under-specified or incorrect mechanisms.

## 2. Problems in the current draft

- **P1** — Only one access path is described. A non-pNFS client (NFSv3)
  reaches the PS directly and never sees a layout.
- **P2** — "Route their I/O through it" overstates the PS role. A
  codec-capable client's writes are client-driven to the real mirror
  set; only a codec-incapable client must write through the PS.
- **P3** — `FFV2_DS_FLAGS_PROXY` has no consumer for move/repair: a pNFS
  client does I/O, handles errors and recalls identically whether its
  DS is real or a proxy.
- **P4** — "No new claim type" is not implementable: `nc_is_registered_ps`
  cannot distinguish the PS's proxy OPEN from a racing or unrelated
  OPEN on the same session, and the PS asserts no intent.
- **P5** — Filehandle handling is unspecified/conflated (see §3).
- **P6** — The proxy_stateid's role was left vague enough that the first
  design draft over-loaded it (see §4.4).

## 3. Two filehandles -- keep them distinct

The single biggest correction from the review. There are **two**
filehandles for a proxied file and they must never be conflated:

- **FH-mds** — the file's filehandle *on the MDS*. The MDS mints it; it
  is the existing `pa_file_fh` carried in the PROXY_PROGRESS work
  assignment. The PS uses FH-mds only to talk to the MDS (PUTFH, OPEN,
  LAYOUTGET against the MDS). The PS treats it as opaque.
- **FH-ps** — the filehandle under which the *PS* serves the file to
  clients: the Scenario-A LOOKUP result and the Scenario-B layout's
  DS-entry filehandle. Only the PS can mint it (filehandle format is
  private to the server that decodes it, RFC 8881 §4.2.3). The PS
  conveys FH-ps to the MDS; the MDS treats it as opaque and copies it
  verbatim into the proxy layout it hands to clients.

The MDS mints FH-mds; the PS mints FH-ps. Neither can mint the other's.

## 4. Proposed design

### 4.1 Three client access modes

A client reaches a proxied file in one of three ways:

- **Direct (non-pNFS).** An NFSv3 client (or any client that cannot do
  pNFS layouts) contacts the PS as an ordinary NFS server: LOOKUP,
  obtain FH-ps, do NFS READ/WRITE. No layout, no flag. The client never
  "discovers proxying".
- **Proxy layout (codec-incapable pNFS client).** A pNFS client that
  cannot speak the file's codec gets an FFv2 layout whose single mirror
  names the PS (addressed by FH-ps); it does DS I/O to the PS.
- **Real-DS layout (codec-capable pNFS client).** A pNFS client that
  *can* encode the file is not proxied at the layout level at all.
  During a migration it gets the composite real-DS layout the draft
  already defines (the L1/L2/L3 external view): it reads and does
  client-driven mirrored writes to the real DSes. The PS does not
  appear in this client's layout; the PS is simply another concurrent
  writer to the same DSes.

The original sentence described only the middle case and wrongly
implied all clients route I/O through the PS (P1, P2).

### 4.2 Acquisition flow

1. **PROXY_REGISTRATION** — the PS registers; it joins the registered
   set. Successful registration also implies the MDS supports
   `CLAIM_PROXY` (see §4.3).
2. **MDS preparation.** The MDS recalls all outstanding pNFS layouts
   for the file (CB_LAYOUTRECALL) **and waits for completion** — every
   holder has LAYOUTRETURNed, or the MDS has revoked the stragglers
   (RFC 8881 §12.5.5.2) — *before* it publishes the in-flight migration
   record. Issuing the recall is not enough; an un-returned layout
   still authorises client DS I/O.
3. **PROXY_PROGRESS assignment.** The PS polls; the MDS selects a PS and
   returns a work assignment carrying the file (`pa_file_fh` = FH-mds),
   the work type, and a freshly minted `proxy_stateid` (§4.4).
4. **OPEN(CLAIM_PROXY)** — see §4.3.
5. **LAYOUTGET** — the PS obtains the composite layout the ordinary
   way: an ordinary LAYOUTGET using the all-zero special layout
   stateid (bootstrap) or its open stateid from step 4. The MDS knows
   the request is for the composite layout because this clientid holds
   an in-flight migration record for the file.

### 4.3 CLAIM_PROXY

`OPEN` gains a new `open_claim_type4` enumerant, `CLAIM_PROXY`, and an
`open_claim4` union arm carrying:

```
struct open_claim_proxy4 {
        proxy_stateid4  ocp_proxy_stateid;  /* assignment correlator */
        nfs_fh4         ocp_ps_fh;          /* FH-ps the PS will serve */
};
```

Properties:

- **Filehandle-based, open-existing.** Like `CLAIM_FH`, it operates on
  the current filehandle: the PS does `PUTFH(FH-mds); OPEN(CLAIM_PROXY)`
  — no parent directory, no component name. (This is also why the old
  `CLAIM_NULL` framing never fit: `CLAIM_NULL` needs a directory FH and
  a name, and the PS has only the file's FH-mds.) `CLAIM_PROXY` MUST
  NOT be combined with `OPEN4_CREATE`.
- **`ocp_proxy_stateid`** is the explicit, generation-specific
  *correlator*: it proves this OPEN is the proxy OPEN for a specific
  assignment, so the MDS never infers proxy intent from
  `(session, FH)`. This is the fix for P4.
- **`ocp_ps_fh`** carries FH-ps in the operand so registering it is
  atomic with the proxy binding.
- **MDS validation.** The MDS MUST verify: `ocp_proxy_stateid` is valid,
  names an outstanding assignment, that assignment was made to this
  clientid/session, and the current filehandle is the assignment's
  file. On failure: `NFS4ERR_BAD_STATEID` for a bad stateid; the
  assignment/FH-mismatch error to be chosen when drafting.
- **Discovery / compatibility.** A PS MUST NOT issue `OPEN(CLAIM_PROXY)`
  without a successful PROXY_REGISTRATION; registration success *is*
  the signal that the MDS supports the claim type (a pre-`CLAIM_PROXY`
  MDS rejects PROXY_REGISTRATION, op 93, first, so the PS never reaches
  the OPEN).
- **Results unchanged.** `OPEN(CLAIM_PROXY)` returns the standard `OPEN`
  reply: an ordinary open stateid, and `open_delegation4` which the MDS
  **MUST** set to `OPEN_DELEGATE_NONE`. The open stateid is used for
  CLOSE and may bootstrap the PS's LAYOUTGET; it is an ordinary stateid
  with its own `other`.
- **Share semantics.** `OPEN4_SHARE_ACCESS_BOTH` / `OPEN4_SHARE_DENY_NONE`.
- **Replay.** `CLAIM_PROXY` defines *no* special replay behaviour. A
  retransmit is absorbed by the NFSv4.1 session replay cache; a genuine
  repeated OPEN by the same open-owner is an ordinary share-state
  operation, exactly as for any other claim type.

### 4.4 proxy_stateid -- a control-plane token only

The `proxy_stateid` is minted by the MDS, fresh per assignment, bound to
the assigned PS's clientid, and delivered in the PROXY_PROGRESS
assignment. Its uses are: the `CLAIM_PROXY` correlator (§4.3), and
PROXY_DONE / PROXY_CANCEL.

It is **not** a layout stateid and **not** a LAYOUTGET argument. It
remains disjoint from the layout value space (a non-PROXY op presenting
it still gets `NFS4ERR_BAD_STATEID`). The earlier idea of presenting it
to LAYOUTGET, or of a "stateid chain" keyed on a shared `other`, is
dropped: RFC 8881 §8.2 makes `other` the field that *distinguishes*
state objects, so it cannot be shared across clients' layout stateids.

### 4.5 Layout gating via the in-flight migration record

While an in-flight migration record exists for file F (keyed on
clientid + file_FH, as the draft already does):

- every layout the MDS issues for F is proxy-shaped — a 1-mirror PS
  layout for a codec-incapable client, the composite real-DS layout for
  a codec-capable client (§4.1);
- the MDS refuses to honour a stale pre-proxy layout stateid for F and
  will not issue a non-proxy layout for F;
- each client's layout stateid keeps its own distinct `other`, per
  RFC 8881 §8.2. There is no shared-`other` chain.

PROXY_DONE retires the migration record; the MDS then recalls and
reissues normal layouts.

### 4.6 The NFS4ERR_DELAY window

From assignment until the PS's `OPEN(CLAIM_PROXY)` registers FH-ps, the
MDS cannot build a proxy layout, so it answers codec-incapable-client
LAYOUTGET with `NFS4ERR_DELAY`; clients retry. The gate is on the
LAYOUTGET path only — direct (non-pNFS) clients are never gated by it,
and the PS may already be serving them.

The window MUST be bounded. If the assigned PS does not issue
`OPEN(CLAIM_PROXY)` within a deadline (tied to the PS's registration
lease — a PS that misses it is also missing PROXY_PROGRESS
heartbeats), the MDS reassigns or abandons the assignment and stops
returning `NFS4ERR_DELAY`; with no PS yet bound it may fall back to a
normal pre-proxy layout. An unbounded `NFS4ERR_DELAY` is a client hang.

### 4.7 Remove FFV2_DS_FLAGS_PROXY

With the migration record as the gate (§4.5) and an explicit work
assignment, nothing consults `FFV2_DS_FLAGS_PROXY`; it is removed.
"Proxied" is MDS control-plane state. The codec-translation text that
currently uses the flag as a per-request layout discriminator is
restated: a codec-incapable client simply receives a layout that names
the PS (addressed by FH-ps) under a codec it supports — the layout
shape itself carries the distinction, no flag needed.

### 4.8 Writes

A codec-capable client writes to the real mirror set directly
(client-driven FlexFiles mirroring); the PS is not a write target for
it. A codec-incapable client (direct, or codec-incapable pNFS) writes
through the PS, which encodes on its behalf. State this per-case, not
as a blanket "route I/O through the PS".

## 5. End-to-end sequence

```
MDS: CB_LAYOUTRECALL of outstanding pNFS layouts for the file
MDS: wait for LAYOUTRETURN / revoke stragglers          <-- completion gate
MDS: publish in-flight migration record; mint proxy_stateid
MDS: select PS; PROXY_PROGRESS assignment -> {FH-mds, work type, proxy_stateid}
MDS: codec-incapable-client LAYOUTGET -> NFS4ERR_DELAY   (window opens)
PS : PUTFH(FH-mds); OPEN(CLAIM_PROXY{proxy_stateid, FH-ps})
MDS: validate; record FH-ps; return open stateid + OPEN_DELEGATE_NONE
PS : LAYOUTGET (ordinary; all-zero or open stateid) -> composite layout
MDS: build proxy layouts; LAYOUTGET DELAY lifts          (window closes)
...  proxy active: layout issuance gated by the migration record
PS : PROXY_DONE
MDS: retire migration record; recall + reissue normal layouts
```

If the PS never reaches `OPEN(CLAIM_PROXY)`, the §4.6 deadline fires:
the MDS reassigns/abandons and stops the `NFS4ERR_DELAY`.

## 6. Decisions recorded (former open items O1-O4)

- **O1** — Three distinct stateids, no chain: the `OPEN(CLAIM_PROXY)`
  open stateid (ordinary; CLOSE + LAYOUTGET bootstrap), the
  `proxy_stateid` (control-plane token), the composite-layout stateid
  (ordinary layout stateid the PS persists for reclaim). None share
  `other`.
- **O2** — `OPEN_DELEGATE_NONE` on a `CLAIM_PROXY` open is **MUST**.
- **O3** — The `CLAIM_PROXY` operand is `open_claim_proxy4`
  (proxy_stateid + FH-ps), one struct, conveyed in the OPEN.
- **O4** — Assimilation (§8) ships as a **separate, later change**.

## 7. Draft sections affected

- Introduction — the "Clients discover…" sentence: rewrite for the
  three access modes (§4.1) and the writes rule (§4.8).
- "No new claim type" subsection (under "Layout Shape During a Proxy
  Operation") — retitle and invert: there IS a new claim type (§4.3).
- MDS-recovery section — it also asserts "no new claim type"; reconcile.
- proxy_stateid section — state it as assignment-issued and
  control-plane-only; keep it disjoint from the layout value space
  (§4.4).
- PROXY_PROGRESS — the assignment record carries `pa_file_fh` (FH-mds,
  already present), work type, and proxy_stateid. No FH-ps field is
  added here; FH-ps arrives via `OPEN(CLAIM_PROXY)`.
- OPEN section — add `CLAIM_PROXY` and `open_claim_proxy4` (§4.3).
- Layout sections (L1/L2/L3, "proxy DS entry") — reconcile with the
  three access modes; gate via the migration record (§4.5).
- Data-server flags — remove `FFV2_DS_FLAGS_PROXY`; restate the
  codec-translation discriminator (§4.7).

## 8. Deferred: assimilation as a type-migration mode

Recorded for continuity; NOT part of this change. Assimilation
(plain file -> RS-coded) is mechanically a type migration ("layout
transition"), not a separate operation. The capturable property is
source mutability for the duration: an immutable-source assimilation
needs no write fan-out and no copier-vs-write convergence; a
mutable-source migration does. This will be a separate change once the
draft's per-instance delta model is normative.
