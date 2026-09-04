# Codec findings

What building the codecs actually caught. Every bug here was invisible to at
least one kind of test, which is the argument for having several.

The codecs themselves are in `codec/go`. Read this before writing another.

## 2.6 What the tests caught

Two bugs, worth recording because they are the argument for the test existing.

**Property-name case.** The parser upper-cased names for matching, which is
correct -- section 3.3 makes them case-insensitive -- but it also stored the
upper-cased form in `ExtensionProperty.key`. Apple writes `X-ABLabel`;
echoing `X-ABLABEL` is legal and is not the same bytes. Fixed by keeping the
raw name for extensions and the folded one for matching.

**Quoted multi-value parameters, and a blind spot in round-tripping.**
`TEL;TYPE="work,voice"` decoded to zero types: the comma was inside quotes, so
the parser kept `work,voice` as one value, which matched no known TYPE and was
dropped. The RFC's own section 6.4.1 example is `TYPE="voice,video"` meaning
two values, so the quotes are transport, not content.

The important part is *why the round-trip test passed anyway*. Decoding
dropped TYPE and encoding therefore never emitted it, so decode -> encode ->
decode was equal while the data was gone. **Round-trip equality proves
stability, not correctness.** `decode_test.go` now asserts specific decoded
values, which is what actually catches symmetric loss. Any future codec needs
both kinds of test.

## 2.7 What the cross-format test caught

`TestJCardCrossFormat` decodes every `.vcf` fixture, re-encodes it as jCard,
decodes that, and requires the two models to be equal. It failed on its first
run, which is the argument for writing it.

**A `tel:` URI was being escaped through jCard.** The text encoder writes a
URI unescaped, correctly -- section 6.4.1's own example is
`tel:+1-555-555-5555;ext=5555` with a bare semicolon, and escaping it changes
the URI. The jCard path escaped every value uniformly, so the semicolon came
back as `\;`. Fixed by threading the value-type identifier through and
leaving `uri` values alone.

A single-format round trip could not have found this: text -> Contact -> text
never escaped the URI, so it stayed stable. It took a *second* encoding of the
same model to expose the disagreement.

**One legitimate difference, recorded rather than fixed.** RFC 7095 section
3.3 requires property names to be lowercase, so `X-ABLabel` crosses into jCard
as `x-ablabel` and cannot come back. The test compares extension keys
case-insensitively for exactly this reason. It is a property of the format,
and any consumer moving vCards through jCard inherits it.

## 2.8 What the iCalendar codec verified

Nothing failed on the first run, which is worth stating plainly: the schema
schema changes made for it were right. What the tests now pin down:

- **All four time forms decode distinctly.** `TestTimeForms` asserts that a
  floating DATE-TIME carries *no* offset, a `Z` value carries a zero UTC
  offset, a `TZID` value keeps its zone id, and an all-day value is a `Date`
  rather than a midnight instant. Before `CalendarTime` three of those four
  were unrepresentable, so this test could not have been written — which is
  the clearest statement of what that change bought.
- **`Recurrence` models a real RRULE.** `FREQ=MONTHLY;COUNT=12;BYDAY=-1FR;BYSETPOS=-1`
  decodes with the ordinal intact. Losing the `-1` would silently turn "the
  last Friday" into "every Friday".
- **`BYMONTH` maps onto `google.type.Month`**, whose enum numbers are 1-12, so
  the range is enforced by the type.
- **DTEND and DURATION together are rejected**, citing section 3.6.1, rather
  than silently resolved one way. That was a named hazard in §2.4.
- **DURATION parsing is not Go's.** Section 3.3.6 has weeks and days, which
  `time.ParseDuration` rejects outright.

Still unbuilt: xCard (6351) and jCal (7265). jCal should be cheap for the same
reason jCard was — it is a re-encoding of a model that now has tests.

## 2.9 jCal, and where the "pure syntax layer" claim stops

jCard was a syntax layer over the same content lines, nothing more. jCal is
not, and the difference is worth recording because it bounds the claim.

RFC 7265 changes three value representations, so `jcal_value.go` translates
them while `decodeProperty` and `contentLines` stay shared:

| | text/calendar | jCal |
|---|---|---|
| date-time, §3.5.5 | `20260615T090000` basic | `2026-06-15T09:00:00` ISO 8601 extended |
| RECUR, §3.5.10 | `FREQ=MONTHLY;COUNT=12` string | a JSON object with typed members |
| GEO, §3.4.3 | `51.5074;-0.1278` | a two-element float array |
| component | `BEGIN:`/`END:` lines | a 3-element `[name, props, subcomponents]` array |

So the rule is: **the semantic layer is always shareable; the value layer is
only sometimes shareable.** jCard happened to reuse vCard's value forms
verbatim, jCal does not, and assuming otherwise would have produced a codec
that emitted iCalendar-basic dates into JSON and passed its own round trip
while being unreadable to every other jCal implementation.

Two details the tests pin:

- **RECUR members keep their JSON types.** `count` is the number 12, not
  `"12"`, while `byday` is the string `"-1FR"` -- which is not a number and
  must not be coerced into one.
- **The type identifier carries DATE-ness.** jCal has no VALUE parameter, so
  `"date"` in the type slot is the only thing distinguishing an all-day value
  from a date-time. `jcalToLine` turns it back into `VALUE=DATE` before the
  shared decoder sees it; without that every all-day event decodes as a
  date-time.

Only xCard (6351) remains, and it is the same shape a third time.

## xCard, and the bug that appeared twice

RFC 6351 reshapes the model more than the other two encodings: a structured
value becomes *named* child elements rather than an ordered array, so the
mapping needs a component-name table -- `<surname>`, `<given>`,
`<additional>`, `<prefix>`, `<suffix>` for N; seven more for ADR -- instead of
an index. There is no VERSION property at all: section 4 puts the version in
the namespace URI.

**The URI-escaping bug came back.** jCard escaped `tel:+1-418-656-9254;ext=102`
into `tel:...\;ext=102`, and xCard did the identical thing for the identical
reason: a codec that escapes every value uniformly does not know that a
`<uri>` value is not TEXT. Two independent codecs made the same mistake, and
in both cases the cross-format test caught it while the format's own round
trip stayed green.

That is worth generalising: **whenever a format distinguishes value types,
the escape rule is per type, not per codec.** The next codec should be
written with that in mind rather than rediscovering it.

**Namespace inheritance, a bug the RFC had nothing to do with.** Setting
`XMLName.Space` on the root alone made Go's `encoding/xml` stamp `xmlns=""` on
every child, un-declaring the namespace for exactly the elements that need it.
The document parsed back correctly here -- our own decoder does not check
child namespaces -- so only reading the emitted XML revealed it. Declaring the
namespace as an attribute fixes it.

**Order independence is tested, not assumed.** XML may present named children
in any order, so `xcardNamedValue` reads them by name into a map rather than
by position. `TestXCardNamedComponentsAreOrderIndependent` feeds
`<suffix>`, `<given>`, `<surname>` in that order and requires the right
mapping; reading positionally would silently swap a family and given name.

The calendar-side findings -- sub-components, and what the VAVAILABILITY codec
turned up -- are in
[`codec-findings-calendar.md`](codec-findings-calendar.md). The split is rule
2's 250-line cap, not a change of subject.
