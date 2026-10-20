# Quality assurance plan

**Project:** obravera-fingerprint-verifier
**Status:** draft, ObraVera implementation evidence for the fingerprint-schemes
discussion in the SPUR evidence working group. The charter requires a second,
independently implemented verifier before a module is stable.

## 1) Purpose and scope

This plan states what a reader — a working-group reviewer, or someone building
the second verifier — can rely on about how this repository is tested and
released.

It is deliberately narrower than the ObraVera house plan it is adapted from.
This repository is ObraVera's public implementation under Apache-2.0, and its
quality bar is set by a different question: **can somebody else reproduce these
results and re-implement against them?** Conventions that would raise the
barrier for an outside implementer are not adopted here, and where this plan
departs from the house standard it says so and why.

`docs/architecture/implementation-notes.md` maps the design onto the code.
Where the code and the notes disagree, that is a defect in one of them and is
raised rather than silently resolved.

## 2) Environments

- **Local:** any Python 3.10–3.12. `pip install -e ".[dev]"` — plain
  setuptools, deliberately, so no toolchain beyond pip is required to reproduce
  a result.
- **CI:** GitHub Actions, `ubuntu-latest`, matrixed on Python 3.10 and 3.12.
- **Runtime dependency:** `iscc-core`, pinned exactly. Nothing else. No network,
  no service, no ledger, no private ObraVera code.

## 3) Requirements to tests

| Requirement | Where it is asserted |
|---|---|
| A record carries the `similarity_only` proposition and no verdict | `tests/test_boundary.py` |
| A record cannot carry grounding, entitlement, ownership or truth claims | `verify.FORBIDDEN_CLAIM_KEYS`, asserted both ways in `tests/test_boundary.py` |
| A foreign proposition is rejected, not ignored | `test_foreign_proposition_is_rejected` |
| Every scheme resolves and scores its six fixture classes | `tests/test_fixtures.py`, `fixtures/fingerprint/*/manifest.json` |
| Scores are recomputed, never read from the manifest | `verify.run_manifest` |
| Binary fixture sources are byte-reproducible | `tools/make_binary_fixtures.py` + `git diff --exit-code` in CI |

The second row is the pattern this repository leans on hardest: it asserts both
that clean output *is* clean and that a deliberately dirtied record *is
rejected*. A guard that is never exercised against a violation is not known to
work.

## 4) CI gates

Required on pull requests to `main`:

- **Conformance** (`ci.yml`), on Python 3.10 and 3.12 — the fixture runner over
  every manifest, then `pytest`, then regeneration of the binary sources
  followed by `git diff --exit-code`. That last step is the strongest gate here:
  it asserts the committed fixtures are exactly what the tooling produces, so a
  fixture cannot drift from its generator unnoticed.
- **QA doc check** (`quality-doc-check.yml`) — this document exists and carries
  its required sections.
- **pre-commit** — ruff lint and format, gitleaks, actionlint, yamllint, and the
  standard hygiene hooks.

Actions are pinned by commit SHA, not by mutable major tag.

## 5) Dependency supply-chain hygiene

Dependabot watches **GitHub Actions only**. The pip ecosystem is deliberately
excluded, and `.github/dependabot.yml` records the reason: `iscc-core` is pinned
exactly because fingerprints must be byte-reproducible from a clean checkout.
Watching it would produce a steady stream of pull requests arguing against a
decision already made, each requiring every `expected_score` to be re-pinned and
re-verified to accept. Upgrading `iscc-core` is a considered act, not a routine
one, and it is done by hand with the fixture diff reviewed.

Actions are the opposite case. Pinning by SHA is what makes them trustworthy and
equally what makes them go stale silently, so Dependabot is the mechanism that
keeps those pins current.

Secrets are scanned by gitleaks in pre-commit. There is no npm dependency tree.

## 6) Release automation

**Deferred, deliberately. There is none, and this section records why rather
than leaving the absence to be read as an oversight.**

The working-group charter holds this module unstable until a second,
independently implemented verifier exists. Publishing versioned artefacts before
then would imply a stability the module does not have, and would invite
downstream pinning to an interface still open to change by the group.

So there is no tag-triggered workflow, no package published to PyPI, and the
version in `pyproject.toml` is not treated as a release channel. Installation is
from source, which is also the form an implementer reading the code wants.

**When this is revisited:** when a fingerprint-schemes module is accepted and a second
verifier exists. The questions to settle then are what a version number means
when the schemes registry is normative, whether the conformance fixtures are
versioned with the code or separately, and whether publication belongs under the
ObraVera organisation or the coalition's.

## 7) Defect triage

- **S1** — a record can carry a claim the boundary rule forbids, or a
  fingerprint is computed wrongly. Both undermine the only proposition this
  module makes. Immediate fix; assume any affected records are unreliable.
- **S2** — a scheme mis-scores, a fixture class is missing, or the runner
  reports a pass it should not. Fix before the next fixture change.
- **S3** — CLI, output formatting, documentation.

A defect in the boundary rule is always S1, however small it looks. The whole
value of the module is that its records mean exactly one thing.

## 8) Security and privacy baseline

- No keys, credentials or personal data in this repository.
- The verifier parses **untrusted input**: candidate files and manifests supplied
  by whoever runs it. Parser robustness is therefore in scope for security
  reports; see `SECURITY.md`.
- No network access at runtime. The verifier reads local files and computes; it
  contacts nothing.
- Fixture material is CC0 or public domain, original where possible, and every
  vector records its `licence`.

## 9) Governance

- This document and the workflow files must stay aligned. A change to the gates
  is a change to this document.
- `docs/architecture/implementation-notes.md` records the design. This plan governs the process.
- Departures from the ObraVera house standard are recorded where they are made —
  in `.pre-commit-config.yaml`, `.github/dependabot.yml` and here — so that a
  future reader can tell a decision from an omission.
- Commits carry DCO sign-off (`git commit -s`).

## 10) Future expansion

- A coverage floor, once the second verifier makes the interface stable enough
  for coverage to mean something.
- Cross-verifier conformance: running both implementations over the same
  fixtures and comparing, which is the check the charter actually wants.
- Release and versioning, per section 6.

## 11) Change log

- 2026-09-07: Initial plan. Records the CI gates, the Dependabot exclusion, and
  release automation as a reasoned deferral.
- 2026-10-20: Prepared for public release; design references point to the
  implementation notes.
