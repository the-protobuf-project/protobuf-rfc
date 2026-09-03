# Conventions

The second half of the working rules. `CLAUDE.md` holds rules 1-8 (structure
and layout) and imports this file; these are the ones about what goes *inside*
a file. Same standing as the others — none is advisory.

## 9. Two identifiers, not one

`uid` is ours: OUTPUT_ONLY, server-assigned, `(google.api.field_info).format
= UUID4` because AIP-148 requires it.

An RFC's own identity property gets a separate `<format>_uid` field --
`vcard_uid`, `ical_uid` -- marked OPTIONAL and IMMUTABLE. It arrives inside
an imported file, is whatever the originating system chose, and must survive
a round trip unchanged or every re-import duplicates the data. Do not
conflate the two.

## 10. AIP naming traps

An RFC's own term for a property is frequently unusable as a field name.
Rename the field, keep the RFC's name in the comment, and add a row to
PLAN.md's naming table — which is the full catalogue of collisions hit so
far, with the resolution for each.

The recurring ones: AIP-122 bans the `_name` suffix (only `display_name`,
`given_name`, `family_name` survive); AIP-140 bans prepositions, so no
`by_day`; AIP-142 reads a bare time unit like `seconds` as a timestamp;
AIP-216 reserves `state` and `status`; and a field called `name` makes
api-linter treat the message as a resource.

## 11. Every RFC citation carries its link

These comments become the generated documentation in every target language,
so a citation without a URL is a lookup the reader does by hand. Cite inline,
and put a reference block above every message and enum naming the RFC title:

```
// FN, section 6.2.1 <https://www.rfc-editor.org/rfc/rfc6350.html#section-6.2.1>.
//
// Reference: RFC 6350 "vCard Format Specification", section 6.2.1.
// https://www.rfc-editor.org/rfc/rfc6350.html#section-6.2.1
```

Anchors are `.../rfc<N>.html#section-<X.Y.Z>`; AIP mentions get the same
treatment, `AIP-131 <https://aip.dev/131>`.

**Verify after any bulk edit.** A regex linking `section 3.3.10` can backtrack
into `section 3.3` and splice a URL mid-number. It happened twice here and is
silent — the file still compiles and lints:

```sh
grep -rnE "#section-[0-9.]+>\.[0-9]" protobuf/   # must return nothing
```
## 12. Two annotations, two jobs

**`google.api.field_behavior`** declares intent, and api-linter requires it on
every field of a resource or request. Use `IDENTIFIER` on a resource `name`,
`OUTPUT_ONLY` for anything the server sets (`uid`, `etag`, timestamps,
`delete_time`), `IMMUTABLE` where a value may be set once, `REQUIRED` only
where the RFC or this API genuinely demands it.

Response and metadata messages are exempt: every field is server-produced, so
annotating them adds noise and AIP does not ask for it.

**`buf.validate`** (protovalidate) enforces. Constraints come from the RFC,
not from taste — `PREF` is 0-100 because section 5.3 says so, `BYHOUR` is
0-23 because section 3.3.10 does. Cite the section in the field comment so
the number can be checked against the source.

Two traps:

- A constraint on a possibly-unset field must carry
  `(buf.validate.field).ignore = IGNORE_IF_ZERO_VALUE`, or an empty string
  fails `string.uuid` and every create request is rejected. This build uses
  `IGNORE_IF_ZERO_VALUE`; `IGNORE_IF_UNPOPULATED` is from a newer release and
  will not compile.
- **CEL in `(buf.validate.message).cel` is not checked by any gate here.**
  `buf build`, `buf lint` and api-linter all pass on a broken expression; it
  fails at runtime. Test message-level rules against real payloads before
  trusting them.

## 13. Don't pre-build

`Contact` carries nine of RFC 6350's ~forty properties. Add the rest when a
consumer needs them, not to complete the table.
