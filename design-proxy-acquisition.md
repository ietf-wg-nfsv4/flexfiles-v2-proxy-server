# Design change: proxy acquisition, OPEN, and layout gating

**Status:** Recorded (v3, 2026-05-22). Two independent protocol-soundness
reviews; v3 incorporates both. Ready to apply to
`draft-haynes-nfsv4-flexfiles-v2-proxy-server.md`.
**Scope:** how a client reaches a proxied file; how a PS binds to a
file; the two filehandles; how OPEN and LAYOUTGET are gated; and the
availability/durability properties of routing I/O through a PS.

Two regimes run through this whole change and must be kept apart:

- **Migration** -- MOVE and REPAIR. The PS runs a *copier*: it reads a
  source mirror set and writes a target mirror set.
- **Translation** -- codec translation for codec-ignorant clients. The
  PS is a live *transcoder*: it decodes on read and encodes on write,
  in place. Nothing is copied; the file's storage geometry is stable.

The presence or absence of a copier is the dividing line. It is why
migration needs the Single-Layout Model (§4.2) and translation does
not (§4.3). Assimilation is deferred (§9).

## 1. Origin

The review began at one sentence in the Introduction:

> Clients discover that a file is being proxied through the normal
> pNFS layout path -- a layout that names the PS with a new
> data-server flag (FFV2_DS_FLAGS_PROXY) -- and route their I/O
> through it while it is active.

## 2. Problems in the current draft

- **P1** -- only one access path is described; a non-pNFS client
  reaches the PS directly and never sees a layout.
- **P3** -- `FFV2_DS_FLAGS_PROXY` has no consumer (see §4.7).
- **P4** -- "No new claim type" is not implementable:
  `nc_is_registered_ps` cannot distinguish the PS's proxy OPEN from a
  racing OPEN.
- **P5** -- filehandle handling is conflated (see §3).
- **P6** -- the proxy_stateid's role was left vague (see §4.6).
- **P2 is RETRACTED.** P2 read "route their I/O through it overstates
  the PS role." It does not. During a migration, routing all client
  I/O through the PS *is* the design, and the correct one (§4.2).

## 3. Two filehandles -- keep them distinct

There are two filehandles for a proxied file:

- **FH-mds** -- the file's filehandle on the MDS. The MDS mints it; it
  is the existing `pa_file_fh` in the PROXY_PROGRESS assignment. The PS
  uses it only to talk to the MDS (PUTFH, OPEN, LAYOUTGET).
- **FH-ps** -- the filehandle under which the PS serves the file to
  clients: the layout's DS-entry filehandle, and the LOOKUP result a
  non-pNFS client obtains from the PS. Only the PS can mint it
  (filehandle format is private to the decoding server, RFC 8881
  §4.2.3); the PS conveys it to the MDS, which treats it as opaque.

## 4. Proposed design

### 4.1 Reaching a proxied file

A client reaches the PS by one of two front doors, independent of
regime:

- **Direct (non-pNFS).** An NFSv3 client (or any client that cannot do
  pNFS layouts) contacts the PS as an ordinary NFS server: LOOKUP,
  obtain FH-ps, do NFS READ/WRITE.
- **pNFS layout.** A pNFS client receives a layout whose DS entry names
  the PS (addressed by FH-ps) and does DS I/O to the PS.

These are two front doors to the same PS. The access path is a
discovery detail and MUST NOT change data-path semantics.

### 4.2 Migration: the Single-Layout Model

During a MOVE or REPAIR, **every** client of the file routes all I/O
through the PS -- whichever front door it used. The PS is the sole
writer to the source and target mirror sets.

This is required for correctness. The PS runs a copier. If clients
wrote the source mirror directly while the PS also wrote it, there are
two uncoordinated writers: the copier can read range R, a client can
then overwrite R, and the copier can then write its stale R back --
silently losing the client's write. With the PS as the sole writer it
serializes client writes against its own copy operations; the race
cannot arise. (The draft's per-instance delta model, which would be
the alternative mediation, is marked informative and "not exercised by
the wire ops in this revision" -- so Single-Layout is the only sound
normative shape.)

The MDS keeps three logical layout records during a migration; only
L3 is client-facing -- see §8.

### 4.3 Translation: the mixed model

Codec translation copies nothing. The file stays in its native
coding; there is no copier and therefore no stale-overwrite race. The
same file is consequently served two ways at once:

- a **codec-capable** client gets a normal layout naming the real
  DSes and does I/O directly; the PS is not involved;
- a **codec-incapable** client gets a layout naming the PS, or reaches
  the PS directly; the PS transcodes.

This coexistence is correct precisely because there is no copier; the
draft already describes it and it stands.

### 4.4 Writes

Migration: all client writes go through the PS (§4.2). Translation:
codec-capable clients write the real DSes directly; codec-incapable
clients write through the PS, which encodes on their behalf.

### 4.5 The CLAIM_PROXY open claim

Regime-independent: a PS binds to a file the same way for migration
and translation. `OPEN` gains a new claim, `CLAIM_PROXY`:

```
const CLAIM_PROXY = 7;          /* extends open_claim_type4 */
struct open_claim_proxy4 {
        proxy_stateid4  ocp_proxy_stateid;   /* assignment correlator */
        nfs_fh4         ocp_ps_fh;           /* FH-ps */
};
/* open_claim4 gains: case CLAIM_PROXY: open_claim_proxy4 ocp_proxy; */
```

- Filehandle-based, open-existing: `PUTFH(FH-mds); OPEN(CLAIM_PROXY)`,
  no directory or name, like `CLAIM_FH`; never with `OPEN4_CREATE`.
- `ocp_proxy_stateid` is the explicit, generation-specific correlator:
  it proves this OPEN is the proxy OPEN for a specific assignment, so
  the MDS never infers proxy intent from `(session, FH)` (fixes P4).
- `ocp_ps_fh` carries FH-ps, bound atomically with the proxy OPEN.
- MDS validation: the stateid is valid, names an outstanding
  assignment, the assignment was made to the calling clientid, and the
  current filehandle is that assignment's file. Failure draws
  `NFS4ERR_BAD_STATEID`.
- Discovery: a PS MUST NOT issue `OPEN(CLAIM_PROXY)` without a
  successful PROXY_REGISTRATION; registration success is the signal
  that the MDS implements the extension.
- Results unchanged: an ordinary open stateid, and `open_delegation4`
  the MDS MUST set to `OPEN_DELEGATE_NONE`. Share
  `OPEN4_SHARE_ACCESS_BOTH` / `OPEN4_SHARE_DENY_NONE`.
- Replay: no special handling -- ordinary OPEN semantics (session
  replay cache for retransmits; open-owner/seqid for re-OPENs).

### 4.6 proxy_stateid -- a control-plane token only

Minted by the MDS, fresh per assignment, bound to the assigned PS's
clientid, delivered in the PROXY_PROGRESS assignment. Used as the
`CLAIM_PROXY` correlator and in PROXY_DONE / PROXY_CANCEL. It is NOT a
layout stateid and NOT a LAYOUTGET argument; it stays disjoint from
the layout value space. The PS obtains L3 with an ordinary LAYOUTGET;
the MDS serves L3 because the calling clientid holds an in-flight
migration record for the file.

### 4.7 Layout gating, and FFV2_DS_FLAGS_PROXY

While an in-flight migration record exists for file F, every
client-facing layout the MDS issues for F names the PS; the MDS
refuses to honour a stale pre-migration layout stateid for F. Each
client's layout stateid keeps its own distinct `other` (RFC 8881
§8.2); there is no shared-`other` chain.

`FFV2_DS_FLAGS_PROXY` is retained, but recorded as a **non-load-bearing
marker**: no client, MDS, or PS code path depends on it. The client
does DS I/O to whatever its layout names; the MDS issues layouts from
its own state; the PS knows its work from its assignment. What
actually distinguishes migration from translation, and tells each
actor what to do, is MDS+PS control-plane state -- never a layout
flag.

### 4.8 The NFS4ERR_DELAY window (migration setup)

From assignment until the PS's `OPEN(CLAIM_PROXY)` registers FH-ps,
the MDS cannot build the client-facing layout, so it answers **every**
pNFS LAYOUTGET for the file with `NFS4ERR_DELAY` -- during a migration
there is no real-DS layout to fall back to. The window MUST be
bounded: if the assigned PS does not `OPEN(CLAIM_PROXY)` within a
deadline (tied to the PS registration lease), the MDS reassigns or
abandons the assignment and stops returning `NFS4ERR_DELAY`. An
unbounded DELAY is a client hang. (Translation, being a standing
arrangement rather than a discrete operation, has no such window.)

### 4.9 Write durability at the PS

The PS MUST NOT acknowledge a client write until every mirror in the
target set is durable. This closes the torn-write-on-crash hole: a PS
crash cannot leave a client write acknowledged but applied to only
part of the mirror set. An acknowledged write is durable everywhere.

### 4.10 Fencing the outgoing PS on reassignment

On PS reassignment the MDS MUST fence the outgoing PS -- REVOKE_STATEID
of its L3 layout stateid, and/or DS-side fencing -- before the
replacement PS's layout goes live. Otherwise a slow write from the old
PS can race the new PS: a two-PS instance of the very write race
Single-Layout exists to prevent.

### 4.11 Availability and throughput

During a migration the PS is a bounded data-path single point of
failure and a throughput funnel: all client I/O for the file, plus the
PS's own copy traffic, traverse one PS, and the usual FlexFiles
mitigation (client-side mirroring across DSes) is unavailable because
the client sees one DS, the PS. This is inherent to Single-Layout and
cannot be designed away; the draft MUST state it in operational
considerations. It argues against migrating very large files without
the deferred multi-proxy / partial-range work.

## 5. End-to-end sequence (migration)

```
MDS: CB_LAYOUTRECALL of outstanding pNFS layouts for the file
MDS: wait for LAYOUTRETURN / revoke stragglers          <-- completion gate
MDS: publish in-flight migration record; mint proxy_stateid
MDS: select PS; PROXY_PROGRESS assignment -> {FH-mds, work type, proxy_stateid}
MDS: every pNFS LAYOUTGET for the file -> NFS4ERR_DELAY   (window opens)
PS : PUTFH(FH-mds); OPEN(CLAIM_PROXY{proxy_stateid, FH-ps})
MDS: validate; record FH-ps; return open stateid + OPEN_DELEGATE_NONE
PS : LAYOUTGET (ordinary) -> L3 composite layout
MDS: build client layouts naming the PS; DELAY lifts      (window closes)
...  proxy active: PS is sole writer; acks only after target durable
PS : PROXY_DONE
MDS: retire migration record; recall + reissue normal layouts
```

If the PS never reaches `OPEN(CLAIM_PROXY)`, the §4.8 deadline fires.

## 6. Decisions recorded

- **O1** -- three distinct stateids, no chain: the `OPEN(CLAIM_PROXY)`
  open stateid, the control-plane `proxy_stateid`, the L3 layout
  stateid. None share `other`.
- **O2** -- `OPEN_DELEGATE_NONE` on a `CLAIM_PROXY` open is a MUST.
- **O3** -- the `CLAIM_PROXY` operand is `open_claim_proxy4`
  (proxy_stateid + FH-ps), one struct in the OPEN.
- **O4** -- assimilation ships as a separate later change (§9).
- **B3** -- write durability: ack only after the target set is durable
  (§4.9).
- **Migration vs translation** -- Single-Layout is a property of
  migration only; translation runs the mixed model (§4.2, §4.3).

## 7. Draft sections affected

- Introduction -- the "Clients discover…" sentence: rewrite for the two
  front doors and the migration/translation regimes.
- "No new claim type" subsection -- already retitled "The CLAIM_PROXY
  open claim" with the XDR (applied).
- MDS-recovery section -- reconcile its "no new claim type" assertion.
- proxy_stateid section -- state it assignment-issued and
  control-plane-only.
- PROXY_PROGRESS -- assignment record carries `pa_file_fh` (FH-mds,
  present), work type, proxy_stateid.
- Layout Shape section -- the "Single-Layout Model" subsection is
  correct for migration and stays; the "Two-Layout State" subsection's
  "L1 -- the active layout for external clients" wording is wrong and
  is corrected per §8; the "D in M2 … concurrent external client
  writes" rationale and the "Late writes during the swap window"
  subsection are rewritten (no client ever writes the source mirror
  directly during a migration -- the PS does).
- PS Failure and Recovery -- add write-durability (§4.9) and
  outgoing-PS fencing (§4.10).
- Operational/availability text -- add the SPOF and throughput
  disclosure (§4.11).
- `FFV2_DS_FLAGS_PROXY` is retained; the prose around it must not
  present it as load-bearing (§4.7).

## 8. MDS layout bookkeeping during a migration

The MDS keeps three logical layout records: L3 is the PS's composite
working view (read-source set M1, write-target set M2) and is the only
record that backs a client-facing layout -- every client of the file
during the migration is served a layout naming the PS, backed by L3.
L1 and L2 are **MDS-internal bookkeeping** -- the pre-migration mirror
set and the post-migration mirror set -- used to construct L3 and to
know what to commit at PROXY_DONE. They are not client-facing layouts.
The draft's current "L1 -- the active layout for external clients"
wording is the error and is corrected.

## 9. Deferred: assimilation as a type-migration mode

Recorded for continuity; not part of this change. Single-Layout's
single-writer property is itself the mutable-source convergence
mechanism, so assimilation of a mutable source no longer depends on
the per-instance delta model becoming normative for write
correctness. Assimilation will be a separate change.
