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
  - buf.build/the-protobuf-project/protobuf-rfc
```

Then `buf dep update`, and import what you need:

```proto
import "protobuf/rfc6350/user/v1/user.proto";
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
buf generate                                              # Go, Java, Python
buf generate --template buf.gen.ts.yaml --include-imports # TypeScript
```

TypeScript is generated separately because protoc-gen-es emits relative
imports for every dependency, so its output has to be self-contained. The
other three resolve dependencies through published runtime packages.

CI then compiles each SDK — build only, nothing is run, packaged or
published. It exists to prove the schema produces code a consumer can
actually compile, which `buf lint` cannot tell you. The per-language
scaffolding CI copies in is in `sandbox/`, so you can reproduce any of it by
hand:

```sh
cp sandbox/go/go.mod gen/go/ && (cd gen/go && go mod tidy && go build ./...)
```

## What is in it

| Package | Resource | RFC |
|---|---|---|
| `protobuf.rfc6350.user.v1` | `User`, `Contact` | 6350 — vCard |
| `protobuf.rfc4519.organization.v1` | `Organization` | 4519 §3.8 |
| `protobuf.rfc4519.group.v1` | `Group` | 4519 §3.5 |
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

`CLAUDE.md` and `docs/conventions.md` hold the working rules — thirteen of
them, all hard. `PLAN.md` indexes the roadmap in `plan/`.

Before opening a PR, run what CI runs:

```sh
buf format --diff --exit-code   # formatting is a gate, not a convenience
buf lint
buf build
buf generate                                              # Go, Java, Python
buf generate --template buf.gen.ts.yaml --include-imports # TypeScript
```

Plus `api-linter` and the 200-line file cap, both of which CI enforces —
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

