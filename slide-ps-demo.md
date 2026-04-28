<!--
SPDX-FileCopyrightText: 2026 Tom Haynes <loghyr@gmail.com>
SPDX-License-Identifier: CC-BY-4.0

One-slide summary of the cross-PS multi-codec PROXY demonstration
for the Friday talk.  Source for the empirical record is reffs
deploy/sanity/run-ps-demo-codecs.sh.  Reproduce locally via:

  cd reffs && make -f Makefile.reffs run-ps-demo-codecs
-->

# Cross-PS Codec Round-Trip via Flex Files v2 PROXY_*

**Topology** (1 MDS + 6 DSes + 2 PSes + 1 client, all on one Docker bridge):

```
                client
                 /  \
                /    \
            PS-A      PS-B           <-- proxy listeners (port 4098 / 2049)
              \      /
               \    /
                MDS                  <-- ffv2 metadata
              / | | \
            DS0 ... DS5              <-- 6 data servers
```

**The demo:** for each codec, the client opens a per-SB path through PS-A,
performs the codec-encoded write of a 96 KiB random payload, then opens the
same filehandle through PS-B and reads it back. `cmp(1)` of the original and
the PS-B-served payload must match exactly.

**Result matrix** (run on editors' infrastructure, current main):

| SB path           | Layout | Codec                    | PASS / FAIL |
|-------------------|--------|--------------------------|:-----------:|
| `/ffv1-csm`       | FF v1  | plain mirror             | **PASS**    |
| `/ffv1-stripes`   | FF v1  | stripe k=6, m=0          | **PASS**    |
| `/ffv2-csm`       | FF v2  | plain mirror, CHUNK      | **PASS**    |
| `/ffv2-rs`        | FF v2  | RS(4,2), CHUNK           | **PASS**    |
| `/ffv2-mj`        | FF v2  | Mojette systematic (4,2) | **PASS**    |

**5 / 5 codecs PASS** byte-exact through cross-PS write/read.

**What this exercises in the proxy-server draft:**

- `PROXY_REGISTRATION` over mTLS (RFC 9289) with allowlist by client-cert SHA-256
- `PROXY_PROGRESS` heartbeat path (no `proxy_assignment4` -- normal-traffic case)
- LAYOUTGET / GETDEVICEINFO / LAYOUTRETURN / LAYOUTERROR forwarding through PS
- `nfs_fh4` opacity: PS forwards the MDS-minted FH verbatim per RFC 8881 §4.2.3

**What this does NOT yet exercise** (deliberately scoped out):

- `PROXY_OP_MOVE` / `PROXY_OP_REPAIR` assignments
- `PROXY_DONE` / `PROXY_CANCEL`
- Codec translation by the PS (the client codec runs end-to-end here)

**Reproducibility:** open-source, AGPL-3.0; `make -f Makefile.reffs run-ps-demo-codecs`.
