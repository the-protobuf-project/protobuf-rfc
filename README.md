# protobuf-rfc

Protobuf schemas for things the IETF already specified — vCard, iCalendar,
LDAP directory entries — shaped to Google's API Improvement Proposals.

Published as a schema module on the BSR. Depend on it and generate for your
own language, or take the SDKs CI builds — Go, TypeScript, Python and Java are
all generated and compiled on every change.

## Using it

Add the dependency to your `buf.yaml`:

```yaml
version: v2
deps:
  - buf.build/the-protobuf-project/rfc
```

Then `buf dep update`, and import what you need:

```proto
import "protobuf/rfc6350/vcard/v1/vcard.proto";
import "protobuf/rfc5545/event/v1/event.proto";
```

### Setting your own Go import path

No file carries a `go_package`, because the right value depends entirely on
where you generate. Set it once in your `buf.gen.yaml`:

```yaml
version: v2
managed:
  enabled: true
  override:
    - file_option: go_package_prefix
      value: github.com/your-org/your-repo/gen
plugins:
  - remote: buf.build/protocolbuffers/go
    out: gen
    opt: paths=source_relative
```

`java_package` *is* set on every file (`io.github.theprotobufproject.*`)
because api-linter requires it (AIP-191). Override it the same way if you need
a different root.

### Generating locally

```sh
buf generate                                                  # Go, Java
buf generate --template buf.gen.ts.yaml --include-imports     # TypeScript
buf generate --template buf.gen.python.yaml --include-imports # Python
```

Go and Java resolve their dependencies through published artifacts, so the
default template suffices. TypeScript and Python each need
`--include-imports`: protoc-gen-es emits relative imports for every
dependency, and the generated Python does `from buf.validate import
validate_pb2`, which no PyPI package exposes — protovalidate vendors that
module privately under `protovalidate/_gen`.

CI then compiles each SDK on Go 1.26, TypeScript 7, Python 3.14 and Java 25 —
build only, nothing is run, packaged or published. It exists to prove the schema produces code a consumer can
actually compile, which `buf lint` cannot tell you. The per-language
scaffolding CI copies in is in `sandbox/`, so you can reproduce any of it by
hand:

```sh
cp sandbox/go/go.mod gen/go/ && (cd gen/go && go mod tidy && go build ./...)
```

## What is in it

| Package | Resource | RFC |
|---|---|---|
| `protobuf.rfc6350.vcard.v1` | `Vcard`, `Contact` | 6350 — vCard |
| `protobuf.rfc4519.organization.v1` | `Organization` | 4519 §3.8 |
| `protobuf.rfc4519.group.v1` | `LdapGroup` | 4519 §3.5 |
| `protobuf.rfc4519.membership.v1` | `Membership` | 4519 §2.17 |
| `protobuf.rfc5545.calendar.v1` | `Calendar` | 5545 §3.4 |
| `protobuf.rfc5545.event.v1` | `Event`, `Recurrence` | 5545 §3.6.1 |
| `protobuf.shared.interchange.v1` | — (service only) | native |

One package per resource, each self-contained: its resource, value objects,
messages and service. Every RFC citation in every comment carries a link to
the exact section, so the generated documentation is clickable in your IDE.

### Validation

Fields carry `buf.validate` constraints taken from the RFCs' own limits —
`PREF` is 0-100 because RFC 6350 §5.3 says so, `BYHOUR` is 0-23 because RFC
5545 §3.3.10 does. Run them with
[protovalidate](https://github.com/bufbuild/protovalidate) in your language of
choice; no server-side code here enforces anything.

`google.api.field_behavior` marks intent — `IDENTIFIER`, `OUTPUT_ONLY`,
`IMMUTABLE`, `REQUIRED` — on every resource and request field.

## Services

Six resource services with the AIP-131..135 standard methods plus AIP-164
undelete, and `Interchange`, which exports and imports any resource type in
its native format through long-running operations.

There is no server here. The `google.api.http` annotations describe the REST
mapping a host should implement; each RPC's comment documents its verb, path,
idempotency and error codes.

## Contributing

`CLAUDE.md`, `docs/conventions.md` and `docs/ontology.md` hold the working
rules — fourteen of them, all hard. `PLAN.md` indexes the roadmap in `plan/`.

Before opening a PR, run what CI runs:

```sh
buf format --diff --exit-code   # formatting is a gate, not a convenience
buf lint
buf build
buf generate                                              # Go, Java, Python
buf generate --template buf.gen.ts.yaml --include-imports # TypeScript
```

Plus `api-linter` and the 250-line file cap, both of which CI enforces —
see `.github/workflows/ci.yml`. There are no helper scripts: the workflow is
the single definition of every gate. No lint rule may be disabled.

## License

Apache License 2.0 — see [`LICENSE`](LICENSE) and [`NOTICE`](NOTICE).

Every `.proto` carries an SPDX header, and CI rejects one that does not:

```proto
// Copyright 2026 The Protobuf Project authors.
// SPDX-License-Identifier: Apache-2.0
```

protoc-gen-go copies that header into the generated `.pb.go`, so the notice
travels with the Go SDK.

The schemas model structures specified by IETF RFCs. Those documents are the
IETF's and are covered by BCP 78; `NOTICE` records the distinction between
them and the original work here.

## Conformance

`codec/go` implements all five wire formats the schemas claim to model —
`text/vcard` (RFC 6350), `application/vcard+xml` (RFC 6351),
`application/vcard+json` (RFC 7095), `text/calendar` (RFC 5545) and
`application/calendar+json` (RFC 7265) — and is the repository's conformance
test. The content-line syntax the two families share lives in
`internal/contentline`. It is deliberately **not** a published library — it exists to answer the
one question `buf lint` and `api-linter` cannot: does a real `.vcf` survive
import and export unchanged?

```sh
buf generate && cp sandbox/go/go.mod gen/go/go.mod
(cd gen/go && go mod tidy)
(cd codec/go && go test ./...)
```

CI runs it in the `Conformance` job. Three kinds of test, and all three are
needed:

- `roundtrip_test.go` — decode → encode → decode is stable.
- `decode_test.go` — specific properties decode to specific values.
  Round-trip equality alone is blind to *symmetric* loss: a property dropped
  on both sides still compares equal.
- `jcard_test.go` — text/vcard and jCard produce the same `Contact`. Two
  encodings of one data model cannot legitimately disagree, and this found a
  URI-escaping bug that neither single-format test could see.
- `ical/*_test.go` — the calendar equivalents, including `TestTimeForms`,
  which asserts that floating, UTC, zoned and all-day values stay distinct.
  That test is the reason `CalendarTime` exists instead of a `Timestamp`.

Each family has a text encoding and a JSON encoding, and the cross-format
tests require both to produce the same message. Two encodings of one data
model cannot legitimately disagree.

