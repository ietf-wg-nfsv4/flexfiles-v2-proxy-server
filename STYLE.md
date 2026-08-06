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

## Conformance

This draft was brought into conformance in `67424a5`. The §11 pre-commit
checks are clean; the only surviving short forms are sanctioned
exceptions:

- `MDS` and `DS` in ASCII sequence diagrams and one table row, where box
  and column width dominate (§3.1).
- `FFv2` in the front-matter `abbrev`, which is document metadata rather
  than prose (§3.2).

Re-run the §11 greps before each commit.
