# protobuf-rfc — working conditions

Protobuf types for things the IETF already specified. Every rule below is a
hard constraint, not a preference. `.github/workflows/ci.yml` enforces the
ones that can be enforced mechanically -- inline, with no helper scripts, so
the workflow is the single definition of every gate. The rest are on you.

## 1. No lint rule may be disabled

Not with an `except` list in `buf.yaml`, not with a `disabled_rules` entry in
an api-linter config, not with an inline `(-- api-linter: ... =disabled --)`
comment. There is no api-linter config file in this repo and there must not
be one.

If a linter objects, the code is wrong. Fix the code.

When two linters genuinely contradict each other, do not except a rule —
narrow one tool's scope so they stop overlapping. That is why `buf.yaml` uses
`BASIC` rather than `STANDARD`: buf STANDARD wants `UsersService` and
`GetUserResponse` wrappers, AIP-131 wants `Users` returning `User`, and no
code satisfies both. buf checks structure, api-linter checks API design, and
every rule in each selected category is enforced.

## 2. 200 lines per file, hard

Comments and blank lines included. Applies to `.proto` and `.md` alike. A
file over the cap gets split, never compressed — do not strip comments to
squeeze under.

The split order for an RFC directory:

| File | Contents |
|---|---|
| `<resource>.proto` | the resource message |
| `types.proto` | value objects and enums it embeds |
| `messages.proto` | every request and response |
| `service.proto` | the service and its RPCs |

A package with no service omits `service.proto`. A resource with no embedded
value objects omits `types.proto`.

## 3. AIP is the convention

api-linter must report zero violations. In practice that means:

- `option java_package`, `java_outer_classname`, `java_multiple_files` on
  every file — api-linter mandates all three (AIP-191)
- **no `go_package`**: the Go import path depends on where a consumer
  generates, so consumers set it through managed mode. Nothing forces it, so
  nothing is committed. The asymmetry with Java is not a preference —
  api-linter requires one and not the other, and rule 1 forbids excepting it
- a comment on every message, field, enum, enum value, service and RPC
- `(google.api.field_behavior)` on every field
- `(google.api.resource)` on every resource, `(google.api.resource_reference)`
  on every field naming one
- `(google.api.http)` on every RPC, `(google.api.method_signature)` on the
  standard methods
- `int32`, never `uint32` (AIP-141)
- messages before top-level enums; services before messages
- a `uid` field must carry `(google.api.field_info).format = UUID4` (AIP-148)

## 4. No cross-package message references

AIP-215 forbids a field from referencing a message in another proto package,
and exempts only `google.*`. A `protobuf.type.v1` common package does not
help — it was tested and is still flagged.

So a type used across packages is a **documented scalar**, not a wrapper
message: `Telephone.uri` is a `string` documented as an RFC 3986 URI. Do not
reintroduce wrapper messages to "fix" this — that is what the rule prohibits.
`Uri`, `ResourceId` and `MediaType` were exactly that and were deleted,
because nothing outside their own package could legally reference them.

**`google.*` is the one exemption, so reach for `google.type` first.** It is
the only way to share a structured value type across packages instead of
copying it into each. Before defining any value message or enum, check what
already exists — `docs/references.md` lists where, and how to read the
definitions locally with `buf export`.

Adopt only where the model matches. `PostalAddress` and `PhoneNumber` were
both rejected: `PostalAddress` is `address_lines` plus region/administrative
area, which cannot hold RFC 6350 §6.3.1's fixed seven multi-valued
components without losing them, and `PhoneNumber` models E.164 plus short
code, which would reject the free-text TEL values §6.4.1 explicitly permits.
Fidelity to the RFC outranks reuse; say in the comment which you chose and
why.

Create an RFC directory only when that RFC defines something with
**behaviour or identity**. An RFC defining a string format contributes a
comment on a field, not a package.

## 5. No custom proto annotations

No custom options, no extending `MessageOptions`, no `protobuf.meta` package.
RFC provenance goes in comments: name the RFC number and the section, on the
message and on each field that maps to a property.

A type with no RFC behind it lives in `shared/v1` and says in its comment why
it exists anyway. A native type must never sit in an `rfc*/` directory, where
the path alone would lend it authority it has not earned.

## 6. Layout

```
protobuf/rfc<number>/<resource>/v1/<file>.proto
    package protobuf.rfc<number>.<resource>.v1
```

One directory per **defining** RFC, one per resource inside it, then the
version. Never a flat directory of every file for an RFC. Documents that
refine or re-encode a type are cited in comments, not given directories.

Each resource directory is a self-contained package holding its resource,
value objects, messages and service — forced by rule 4, since a package must
contain everything it references.

A direct consequence: **one service per resource.** `Groups` and
`Memberships` are separate services, so creating a group with members is two
calls and is not atomic. That cost is accepted; do not avoid it by merging
packages.

The buf module is rooted at the **repository root**, not at `protobuf/`, so
that `protobuf/` is itself the first package segment and
`PACKAGE_DIRECTORY_MATCH` holds with no redundant directory inside it.
## 7. Cross-cutting behaviour is one generic service, not a method per resource

If a capability applies to resources in general — export, import, audit,
search — it goes in `shared/v1` as one service keyed on `resource_type`, the
string each resource already declares in `google.api.resource`. It does not
get copied onto each resource's service.

Resource services stay at exactly the AIP-131..135 standard methods. Adding a
custom method to one needs a reason that applies to that resource and no
other.

Generic services return resource **names**, not resources — naming a concrete
message type would be a cross-package reference. That is a real cost and it
is accepted; see rule 4.

## 8. Every resource has full CRUD, and every parent is a real resource

A message with `google.api.resource` gets all five standard methods — `Get`,
`List`, `Create`, `Update`, `Delete` — in its package's `service.proto`. A
read-only resource still declares them and rejects writes at runtime.

**And check the pattern.** If a pattern is
`calendars/{calendar}/events/{event}`, `Calendar` must itself be a declared
resource with its own package and service. A parent no resource declares is a
collection nobody can create or delete, and no linter sees it — api-linter
validates the reference string, not whether its target exists. That gap
survived four clean lint runs here. Audit with:

```sh
grep -rh -A3 "google.api.resource) = {" protobuf/ | grep -E "type:|pattern:"
```

## Rules 9-13

Identifiers, AIP naming traps, RFC links, the two annotation systems, and
the don't-pre-build rule are in `docs/conventions.md`, imported below. They carry the same weight as the
rules above; the split exists only because this file has a 200-line cap of
its own (rule 2).

@docs/conventions.md

## Where to look things up

`docs/references.md` is a checked link index: the AIPs and their linter rule
pages, `google/type` and the rest of googleapis, buf and protovalidate docs,
and every RFC this schema models. **Follow the link rather than working from
memory** — section numbers and rule semantics have both been wrong here when
guessed.

@docs/references.md

## Before you finish

Run what CI runs:

```sh
buf format --diff --exit-code
buf lint
buf build
```

plus `api-linter` over `protobuf/**/*.proto` and the 200-line cap. All of it
is inline in `.github/workflows/ci.yml` -- read the steps there rather than
trusting this list to stay current.

Note that the VS Code Google API Linter extension reads
`workspace.protobuf.yaml`, not the CLI's arguments, so it may report
differently. The script is the authority.
