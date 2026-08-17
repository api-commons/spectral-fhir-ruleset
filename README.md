# @api-common/spectral-fhir-ruleset

A curated, **owned**, **grounded** [Spectral](https://github.com/stoplightio/spectral)
ruleset for **HL7 FHIR R4 and R5** RESTful APIs — check that a FHIR-shaped API is
actually FHIR.

An [API Commons](https://apicommons.org) tool.

## Adopt it

```yaml
# .spectral.yml
extends:
  - https://raw.githubusercontent.com/api-commons/spectral-fhir-ruleset/main/fhir.yaml
```

Built-in Spectral functions only — no custom JavaScript — so it runs anywhere Spectral
runs, including in a browser.

## What it is for

There is a large gap between *returning Patient objects* and *being a FHIR server*, and
most of it is invisible until an integration fails. This ruleset governs an OpenAPI that
**describes** a FHIR API and checks the HTTP surface FHIR actually specifies:

| Rule | Severity | Grounding |
| --- | --- | --- |
| `fhir-media-type` | warn | §3.1.0.1.9 — `application/fhir+json`, not plain JSON |
| `fhir-request-media-type` | warn | §3.1.0.1.9 — request bodies too |
| `fhir-error-operationoutcome` | warn | OperationOutcome, not a bespoke error envelope |
| `fhir-capabilities-endpoint` | warn | §3.1.0.10 — a server **SHALL** expose `/metadata` |
| `fhir-resource-type-path-segment` | error | §3.1.0.1.1 — `/Patient`, never `/patients` |
| `fhir-instance-id-parameter` | info | §3.1.0.1.1 — the Logical Id is called `id` |
| `fhir-read-etag` | warn | §3.1.0.5 — servers SHOULD always return an ETag |
| `fhir-etag-weak` | info | §3.1.0.1.3 — versionId is a **weak** validator, `W/"3141"` |
| `fhir-update-if-match` | warn | §3.1.0.5 — optimistic locking |
| `fhir-update-version-conflict` | warn | §3.1.0.4 — document 409 or 412 |
| `fhir-create-location` | info | §3.1.0.4 — `Location` on 201 |
| `fhir-search-returns-bundle` | warn | §3.1.0.7 — a searchset Bundle, not a bare array |
| `fhir-format-parameter` | hint | §3.1.0.1.11 — the `_format` general parameter |
| `fhir-smart-scopes` | warn | SMART App Launch scope syntax |
| `fhir-version-declared` | info | §3.1.0.1.10 — say which release you implement |

### The three that catch the most

**`fhir-resource-type-path-segment`.** FHIR resource types are case-sensitive PascalCase
singular names from a closed list. `/patients` and `/Patients` are both wrong, and both
are exactly what a team writes when bolting FHIR-shaped payloads onto REST conventions.
The ruleset embeds the **178-type union of the R4 and R5 resource lists**, harvested from
`hl7.org/fhir/{R4,R5}/resourcelist.html`.

**`fhir-error-operationoutcome`.** OperationOutcome is FHIR's problem-details equivalent.
An API that errors with `{"error": "..."}` forces every client to carry two error parsers.

**`fhir-read-etag` / `fhir-update-if-match`.** FHIR's concurrency story is ETag plus
If-Match, and the versionId is a **weak** validator — `W/"3141"`. A strong-ETag example
in a FHIR API description is wrong, and it gets copied.

## What this does not do

**It does not validate FHIR resource instances.** Profile and IG conformance needs
StructureDefinitions, terminology servers, and the resource graph — none of which exist
inside an OpenAPI document. Use the [HL7 validator](https://validator.fhir.org/) for
that. This ruleset checks the *HTTP surface*, which the validator does not.

**It cannot catch a version mismatch.** The resource list is the R4 ∪ R5 union, because
flagging a valid R5 resource on an R5 API would be a false positive. The cost is that an
R4 API using an R5-only resource type passes. Pin the release in `info.description` and
review by hand.

**It does not check search parameter semantics.** Whether `?subject=Patient/123` is a
valid search for a given resource type depends on the SearchParameter definitions, not on
the OpenAPI.

## Tests

```
npm install
npm test
```

The harness asserts every declared rule fires on `fixtures/noncompliant.yaml`, that
`fixtures/clean.yaml` is completely silent, and that no rule throws. The first assertion
caught a rule that could never fire; the second caught two rules aimed at the wrong node.

## Validated against a real server

Run against the public [HAPI FHIR R4 server](https://hapi.fhir.org/baseR4)'s own
generated OpenAPI — **2,973 paths**:

```
292  fhir-read-etag
146  fhir-update-if-match
146  fhir-update-version-conflict
  1  fhir-format-parameter
```

**Zero false positives on resource types.** Getting there mattered: the first run flagged
64 paths, and every one was a bug in this ruleset rather than in HAPI — `Bundle` had been
wrongly excluded from the resource list, three real types (`EffectEvidenceSynthesis`,
`Parameters`, `Requirements`) had been dropped by hand-editing the pattern, the root path
`/` was not allowed, and vendor operations with dots (`$hapi.fhir.reindex-status`) did not
match. The pattern is now generated from the harvested list rather than typed.

The 584 remaining findings are **real**: HAPI's generated OpenAPI does not document the
`ETag` / `If-Match` / 409-412 concurrency contract, even though the running server
implements it. Expect this on any server-generated FHIR OpenAPI — the generator omits
headers, and that omission is exactly what a consumer reads.

## License

[Apache-2.0](LICENSE) — Copyright 2026 API Commons (Kin Lane).
