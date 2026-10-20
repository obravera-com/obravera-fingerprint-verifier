# Implementation notes

This document maps the verifier's design onto the code, and records the
decisions an implementer has to make for itself — the things a second verifier
has to choose, and where this one chose.

If this document and the code disagree, that is a defect. Raise it rather than
editing either to match.

## Where each part of the design lives

| Concept | Code |
|---|---|
| Bounded proposition | `verify.PROPOSITION`, `verify.FORBIDDEN_CLAIM_KEYS`, `verify.validate_record()`; asserted in `tests/test_boundary.py` |
| Scheme registry | `schemes.py` — one `Scheme` per row, plus `schemes.get()` |
| Evidence record | `verify.MatchResult.record()` |
| Trust-policy fields | `threshold` on `match()`; `MatchResult.verdict` computes but does not embed |
| Conformance fixtures | `verify.FIXTURE_CLASSES`, `run_manifest()`, `run_tree()`; vectors under `fixtures/fingerprint/` |

## The shape of a scheme

A registry row and its two operations are the same object. `Scheme` carries the
published metadata — `scheme_id`, `algorithm_ref`, `implementation_ref`, `unit`,
`similarity_measure`, `known_defeats`, `false_positive_profile` — alongside
`fingerprint(bytes) -> str` and `score(str, str) -> float`.

Keeping them together is deliberate: the registry row is what a consumer reads
to decide whether a score means anything, and separating it from the code that
produces the score is how the two drift apart. `obravera-fingerprint-verifier
registry` prints the rows, so what is published is read from the same object
that computed the result.

Adding a scheme means adding a `Scheme` and its six fixture classes. Nothing
else in the code needs to know about it.

## Choices left to implementers, and what this verifier does

**The record carries no verdict.** `MatchResult.verdict` exists and is computed
as `score >= threshold`, but `record()` does not include it. The CLI and the
fixture runner display it; the record does not carry it. That separation is the
boundary rule in mechanism rather than in prose — a consumer reaches their own
conclusion under their own trust policy, and nothing in the record does it for
them.

**Block mode picks the best-matching reference block.** With `unit="block"`,
`match()` fingerprints each block of the *reference* and keeps the highest
score, reporting which block in `unit_ref` (`"block:1"`). Text schemes only;
binary schemes raise. A different implementation could reasonably choose
worst-match or an average — nothing yet settles it — so `unit_ref` exists to
make the choice visible in the output rather than implicit.

**Winnowing parameters are fixed at k=25 characters after normalisation, w=4**,
with BLAKE2b-64 k-gram hashes. These are in the registry row's
`implementation_ref`, so a second implementation can match them exactly rather
than inferring them from scores.

**Scores are recomputed on every run.** `run_manifest()` never reads a
fingerprint or a score from a manifest and trusts it; `expected_score` is a pin
to compare against, produced only by `tools/pin_expected_scores.py` and
re-pinned deliberately with the diff reviewed. A manifest is therefore a claim
being tested, not an input being believed.

**A manifest missing any of the six fixture classes fails.** `FIXTURE_CLASSES`
is exhaustive and includes both negatives, `negative_unrelated` and
`negative_known_defeat`. A scheme that only demonstrates its successes has not
demonstrated anything.

## CLI surface

Three subcommands, in `__main__.py`:

- `fixtures <root>` — run every manifest under a tree. **Exit 1 if any vector
  fails**, 0 otherwise, so it works as a CI gate.
- `match <scheme_id> <reference> <candidate>` — score one pair and print the
  record.
- `registry` — print the registry rows, including each scheme's
  `known_defeats` and `false_positive_profile`.

The version reported in records comes from installed package metadata, falling
back to `0.0.0+checkout` when running from an uninstalled tree — so a record
produced from a working copy is visibly not a record produced from a release.

## Determinism

`tools/make_binary_fixtures.py` regenerates the binary sources deterministically
from a fixed seed. CI regenerates them and then runs `git diff --exit-code`, so
a fixture that no longer matches its generator fails the build rather than
quietly becoming the new truth.

This is also why `fixtures/` is excluded from the whitespace-fixing pre-commit
hooks. Appending a newline to a text source changes its Content-Code, changes
its similarity score, and silently invalidates every pinned expectation. See the
comment in `.pre-commit-config.yaml`.

## For a second implementer

The parts that must agree for two verifiers to be comparable, in order of how
easily they go wrong:

1. **Fingerprint bytes.** Same input, same `scheme_id`, same fingerprint string.
   Start here; everything else is downstream.
2. **Normalisation.** For text schemes, what `_normalise()` does before k-gramming
   — differences here look like scoring bugs.
3. **The similarity measure.** Documented per row in `similarity_measure`.
4. **Fixture classes.** All six, both negatives, or the comparison has gaps.
5. **The record shape**, and the fields it must not contain.

Running both implementations over `fixtures/fingerprint/` and comparing is the
check the charter actually wants; the fixtures are structured for it.
