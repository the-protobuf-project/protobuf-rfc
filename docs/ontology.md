# Ontology

The third file of working rules. `CLAUDE.md` holds rules 1-8 (structure and
layout), `docs/conventions.md` holds 9-13 (what goes inside a file), and this
one holds rule 14: which RFC in a lineage the schema is actually built on, and
which way data moves between them. Same standing as the others.

## 14. The newest published RFC is the model; the older ones are formats

Most things this schema covers have a lineage — a modern JSON data model and
the text format it grew out of. Build the **newest published** RFC first. The
older one is the secondary run, and it is not going away.

| Concept | Canonical | Legacy | Conversion | State |
|---|---|---|---|---|
| Contact | [RFC 9553](https://www.rfc-editor.org/rfc/rfc9553.html) JSContact, as amended by [9982](https://www.rfc-editor.org/rfc/rfc9982.html) | [RFC 6350](https://www.rfc-editor.org/rfc/rfc6350.html) vCard | [RFC 9555](https://www.rfc-editor.org/rfc/rfc9555.html) | build |
| Identity | [RFC 7643](https://www.rfc-editor.org/rfc/rfc7643.html) SCIM, +9865, +9967 | [RFC 4519](https://www.rfc-editor.org/rfc/rfc4519.html) LDAP | none specified | build |
| Calendar | [RFC 8984](https://www.rfc-editor.org/rfc/rfc8984.html) JSCalendar | [RFC 5545](https://www.rfc-editor.org/rfc/rfc5545.html) iCalendar | draft only | **deferred** |
| Scheduling | none published | [RFC 5546](https://www.rfc-editor.org/rfc/rfc5546.html) iTIP | — | legacy only |

### "Published" is literal

A draft is not an RFC, however imminent. JSCalendar 2.0 obsoletes RFC 8984
and has been through IESG review for months without a number; the
iCalendar↔JSCalendar conversion is also still a draft. So the calendar
lineage is **deferred entirely** rather than built twice — an 8984 `v1`
would be a major version of a spec that was already superseded when it
shipped.

Deferring is the default when a lineage is in flux. Check the datatracker
before assuming a "modern" RFC is settled.

### Legacy is not deprecated

Both layers get the full AIP-131..135 surface. Rule 8 is unconditional: a
legacy package keeps its resource, its service and its CRUD, and a consumer
holding vCards keeps a vCard API. Nothing here licenses deleting a package
because a newer RFC exists.

What the legacy layer *does* pick up is the updates that make conversion
lossless. RFC 9554 exists precisely so vCard can carry JSContact's data —
PRONOUNS, GRAMGENDER, SOCIALPROFILE, the eleven new ADR components — and
without it the round trip drops fields. Catching the legacy model up is part
of adopting the canonical one, not separate work.

### Naming: the canonical layer takes the plain name

When both layers of a lineage define the same thing, the canonical one gets
the unqualified name, and the legacy one is named for **what its own RFC calls
the object** — not for the RFC number.

| | Canonical | Legacy |
|---|---|---|
| message | `User` | `Vcard` |
| type | `protobufrfc.dev/User` | `protobufrfc.dev/Vcard` |
| pattern | `users/{user}` | `vcards/{vcard}` |
| singular / plural | `user` / `users` | `vcard` / `vcards` |

RFC 6350's object is a vCard, so the resource is `Vcard` and nothing more —
`VcardUser` repeats itself, because the format already names the thing. RFC
4519's group becomes `LdapGroup` on `ldapGroups/{ldap_group}`: `Ldap` alone
names no object, so there the noun is doing work.

The directory follows the resource (rule 6). `Vcard` lives in
`protobuf/rfc6350/vcard/v1`, not in one still called `user`.

All four fields move together — AIP-123 ties the resource type to its message
name and the pattern's collection and final segments to the declared plural
and singular — so renaming a resource is never a one-line change.

Two shapes were tried against the linter and fail. A bare number as its own
segment, `groups/6350/{group}`, puts a literal in an identifier slot and
breaks `resource-name-components-alternate`; `rfc6350/groups/{group}` breaks
the same rule. If a number ever must appear it belongs inside an identifier,
and note that a digit is not a word boundary to the linter: a singular like
`rfc6350User` expects `{rfc6350user}`, with no underscore.

Only what actually collides is renamed. A legacy resource with no canonical
counterpart keeps its plain name, and a child of a renamed parent takes the
new parent segment — `ldapGroups/{ldap_group}/memberships/{membership}` —
without being renamed itself.

### The pipeline is directional at rest

```
   front (either)                        back (canonical)
   ─────────────────                     ────────────────
   vCard / jCard / xCard  ──decode──▶    JSContact
   iCalendar / jCal       ──decode──▶    JSCalendar   (when published)
   LDIF                   ──decode──▶    SCIM

                          ◀──encode───   served back in either
```

Either representation may arrive, and either may be served. What is **stored**
is always the newest. Saving a vCard therefore reads back as JSContact, and
that is the designed behaviour, not a lossy accident — say so in the service
comment rather than leaving a consumer to discover it.

The corollary is that the canonical model must be able to hold everything the
legacy one can. Where it cannot, the gap is documented on the field, and
`ExtensionProperty` (see [`adding-data.md`](adding-data.md)) carries the
remainder rather than dropping it.

### Every codec is bidirectional

`can_export` and `can_import` are separate flags on `Codec` for a reason, but
a codec that only decodes is an unfinished codec. A cross-lineage pair —
`vcard ↔ jscontact` — is a codec like any other, not a second parser bolted
onto the first.

**Where the IETF has specified the conversion, follow it.** RFC 9555, as
updated by RFC 9982, maps vCard to JSContact element by element, including
`JSPROP` and `vCardProps` for elements neither side knows. Do not invent a
mapping that a published RFC already defines.

**Where it has not, say so.** There is no specified SCIM↔LDAP conversion, so
that mapping is ours, and every message it touches must carry a comment
admitting it is a local decision rather than a citation. An invented mapping
presented as an RFC-derived one is the worst outcome available here.

### Rule 4 still binds

Canonical and legacy are **peer packages that import nothing from each other**.
There is no base layer, no shared ontology package, no common type module —
AIP-215 forbids the cross-package reference, and this repository has already
built that layer once and deleted it (`Uri`, `ResourceId`, `MediaType`).

The ontology lives in three places, none of them the import graph:

- the **direction of the codecs**, which is executable
- the **comments**, which name the lineage and the conversion RFC
- the **table above**

A duplicated value type across a canonical and a legacy package is the
expected cost, the same tax rule 4 already charges everywhere else.

### What this rule does not license

- **Building a draft.** See above.
- **Deleting or deprecating a legacy package.** See above.
- **Building the canonical model of a lineage nobody has asked for.** Rule 13
  still applies: newest-first orders the work, it does not authorise all of it.
