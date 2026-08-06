# Drafting style guide

The style guide for this draft family is maintained in one place, in the
base specification's repository:

**<https://github.com/ietf-wg-nfsv4/flexfiles-v2/blob/main/STYLE.md>**

It applies to all three drafts:

- `draft-haynes-nfsv4-flexfiles-v2`
- `draft-haynes-nfsv4-flexfiles-v2-proxy-server` (this repository)
- `draft-haynes-nfsv4-flexfiles-v2-delta-writes`

Keeping a single copy is deliberate: three copies would drift, and the
rules were derived from one editorial pass over the base draft.

## Known gaps in this draft

This draft predates the editorial pass and has not been swept yet. As of
the guide's writing:

| Rule | Occurrences here |
|---|---|
| `MDS` in prose (§3.1) | 9 |
| `DS` / `DSes` in prose (§3.1) | 4 / 5 |
| `FFv2` in body prose (§3.1) | 11 |
| Markdown emphasis (§5.1) | 16 |
| British spellings (§2) | a few |
| `==` in prose (§8) | 1 |

Counts are raw matches; some will be legitimate exceptions (wire
identifiers, table cells, ASCII artwork). Check against the exceptions in
the guide before changing anything, and re-run the pre-commit greps in
§11 afterwards.
