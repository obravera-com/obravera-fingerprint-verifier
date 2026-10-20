# Security policy

## Reporting a vulnerability

Email **security@obravera.com**, or open a private GitHub security advisory on
this repository. Please do not open a public issue for a security problem. We
aim to acknowledge within five working days.

If the issue concerns the Content Telemetry standard itself rather than this
verifier, it belongs with the SPUR Coalition rather than here; say so and we
will route it.

## Scope

This repository is a reference verifier and a set of conformance fixtures. It
holds no keys, no credentials and no personal data, and makes no network
requests at runtime.

**In scope:**

- **Parser robustness.** The verifier reads untrusted input by design —
  candidate files, reference files and manifests supplied by whoever runs it. A
  crash, hang or excessive memory use on a malformed input is a valid report.
- **Anything that lets a record carry a claim the boundary rule forbids.** The
  only proposition a `fingerprint_match` record may make is `similarity_only`.
  A way to emit a record asserting grounding, entitlement, ownership or truth
  defeats the module's entire purpose and is treated as a security issue, not a
  correctness one.
- **Fingerprint or scoring errors** that would cause a verifier to report a
  match where none exists, or the reverse.

**Not in scope:**

- The strength of the fingerprint schemes themselves against deliberate evasion.
  Each registry row publishes its `known_defeats` precisely because these
  schemes are defeatable; that is documented behaviour, not a vulnerability.
  `obravera-fingerprint-verifier registry` prints them.
- Similarity being a weak signal of provenance. It is, and the module says so:
  a score is not a verdict, and a verdict is the consumer's conclusion under
  their own trust policy.

## Dependencies

The only runtime dependency is `iscc-core`, pinned exactly so that fingerprints
are byte-reproducible from a clean checkout. Dependabot deliberately does not
watch it — see section 5 of `docs/qa/QUALITY-ASSURANCE.md`. A security issue in
`iscc-core` is therefore handled by hand: upgrade, re-pin the expected scores,
review the fixture diff, and record the change.

Report a vulnerability in `iscc-core` upstream as well as here.

## Fixture material

All fixture material is CC0 or public domain, original where possible, and every
vector records its `licence`. If you believe a fixture carries material it
should not, report it through the channel above and it will be replaced.
