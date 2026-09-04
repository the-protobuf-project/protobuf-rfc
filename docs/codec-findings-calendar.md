# Codec findings — the calendar codecs

Continues [`codec-findings.md`](codec-findings.md), which holds the vCard-side
findings and the general lessons. Split out under rule 2's 250-line cap.

## Sub-components, and a bug that had been silently corrupting data

Extending the iCalendar codec to ATTENDEE, ORGANIZER and VALARM exposed the
worst bug found so far, and it had been present since the codec was written.

**A VALARM's properties were being decoded into the Event.** The decoder
tracked `inEvent` as a boolean, so once inside a VEVENT it stayed inside --
`BEGIN:VALARM` did not change that. A reminder's `DESCRIPTION:Starts in 15
minutes` therefore *overwrote the event's own description*, and its ACTION and
TRIGGER landed in the event's extensions.

Every test passed. The round trip was stable because the corruption was
symmetric, and the cross-format test agreed for the same reason. Neither could
see it because no fixture had a sub-component: RFC 5545 section 3.6 nests
components, and nothing had exercised the nesting.

The fix is a component *stack* rather than a boolean, and it also lets the
decoder reject `END:VEVENT` closing a `BEGIN:VALARM`.

**The same bug existed in jCal, in mirror image.** `EncodeJCal` also tracked a
boolean, so it flattened VALARM properties up to the calendar level, producing
a document no other implementation could read. And `DecodeJCal` never looked
at a component's third element, so alarms were dropped entirely. Both were
invisible until a fixture had an alarm in it.

**Two asymmetries fell out of the same fixture:**

- `TRIGGER;RELATED=START` decoded to `START` but re-encoded to nothing,
  because the encoder only emitted `RELATED` for `END`. Section 3.2.14 makes
  START the default, so omitting it is *legal* -- but decoding it and then
  dropping it loses a value the model was holding.
- An absolute `TRIGGER` is marked by `VALUE=DATE-TIME` in text/calendar and by
  the type identifier in jCal, which has no VALUE parameter. Keying only on
  the parameter made every absolute trigger fail to parse as JSON. The form is
  now decided by the value: a duration always starts with `P`, a DATE-TIME
  never does.

The generalisation: **a fixture that exercises no nesting cannot catch a
nesting bug, however many encodings agree on it.** Coverage of the *shape* of
a document matters as much as coverage of its properties.

Extending `Contact` past its original nine properties caught two more bugs,
one of them in a fixture rather than in code -- see
[`codec-findings-contact.md`](codec-findings-contact.md).

## What the VAVAILABILITY codec confirmed

Nothing failed on its first run, which is worth stating plainly -- but the
tests were then checked against a deliberately broken decoder before being
trusted. Routing AVAILABLE's properties to its parent, the same shape as the
old VALARM bug above, fails three of them. A test that has never been seen to
fail is not yet evidence of anything.

## The encoder was non-deterministic, and three tests missed it

The VAVAILABILITY codec shipped with `ExtensionProperty.parameters` typed as a
repeated `"NAME=value"` string, filled by ranging over the parser's parameter
map. **Go randomises map iteration**, so one input produced *four distinct
encodings across twenty runs*.

What makes this worth recording is how thoroughly the existing tests failed to
notice:

- **Round-trip passed.** Every individual encoding decoded back correctly.
  Parameter order does not change meaning, so decode -> encode -> decode was
  stable — it just was not *the same bytes twice*.
- **Value assertions passed.** They checked which parameters survived, never
  the order.
- **Mutation testing passed.** Breaking the sub-component dispatcher failed
  three tests, which is what gave confidence in the suite. Determinism was
  simply not among the properties any of them asserted.
- **The fixtures could not have caught it.** No extension in them carried more
  than one parameter, and with one parameter a map has only one ordering.

The fix was to stop diverging: every other `ExtensionProperty` in this schema
types `parameters` as `map<string, string>`, and both the ical and vcard
encoders already sort the keys before writing, with the comment *"Go map order
is not deterministic; output must be."* The new codec had reinvented the
problem those lines exist to prevent.

**Two lessons.** A property no test asserts is not protected by test count,
coverage, or mutation score — and reimplementing something the codebase
already solves is how a fixed bug comes back. Read the neighbouring
implementation before writing a new one.

`TestEncodingIsDeterministic` now encodes the same input twenty times and
requires one distinct result.

## RFC 9073 broke jCal, and a hardcoded component name was why

Adding RFC 9073's three components turned up two bugs, both in code that had
been passing for months.

**The jCal encoder special-cased VALARM.** It walked the content lines with a
component stack, but only VALARM opened a nested frame; every other
BEGIN/END block had its properties appended to whichever list was current.
So a PARTICIPANT's UID, SUMMARY and its nested VLOCATION were all flattened
into the *calendar's* top-level property array. The output still parsed as
JSON and still round-tripped through our own decoder, because the decoder had
the mirror-image assumption — it looked only for `valarm` in the third
element and skipped everything else.

`TestJCalCrossFormat` caught it because it compares the model from text
against the model from jCal, and those two disagreed. A single-format round
trip could not have: both sides were wrong in the same direction, which is
the third instance of that exact shape recorded in these files.

The fix was to stop enumerating component names. jCal's structure is
recursive — every component is `[name, properties, subcomponents]` — so both
encoder and decoder now build and walk the tree generically. **A codec that
lists the components it knows will be wrong the next time a component is
added; one that handles the shape will not.**

**A URI parameter was emitted unquoted.** `STRUCTURED-DATA`'s SCHEMA
parameter is a URI, so it always contains a colon, and RFC 5545 section 3.1
requires such a value to be quoted. Emitting it bare made the parser take
that colon as the property's value separator: the schema decoded as `https`
and the rest of the URI became the payload. The same flaw was in the RFC 9253
LINK encoder, where LINKREL takes a URI for an extension relation.

Caught on the test's first run — and worth noting the *test* was wrong twice
before the code was right: the fixture originally wrote SCHEMA unquoted
(non-conforming input), and the assertion then failed to account for section
3.1 line folding splitting the quoted value across a CRLF. A test that fails
is not yet evidence the code is broken.
