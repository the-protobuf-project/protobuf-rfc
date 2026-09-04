# Codec findings — extending Contact

Continues [`codec-findings.md`](codec-findings.md). `Contact` grew from nine
properties to nineteen -- NICKNAME, URL, ROLE, IMPP, LANG, GEO, TZ, RELATED,
plus BDAY and ANNIVERSARY, which had been typed but never wired into any
codec at all. Two bugs, neither in a new code path a review would check
first.

## A stale fixture became a negative test

`testdata/extensions.vcf` had BDAY, GEO and CATEGORIES lines written to prove
they survived as `ExtensionProperty` -- including a `\,` inside GEO's `geo:`
URI, because a generic extension escapes its comma-separated values on the
way out. Once those three properties graduated to typed fields, the fixture
silently started asserting the wrong thing: a real `geo:` URI is never
escaped, so the fixture was testing an input no real vCard produces.

`TestUnmodelledPropertiesSurvive` passed throughout. It only checks that
*some* extensions survive, not which ones, so it had nothing to say about
three of its six properties quietly changing meaning underneath it.

**Promoting a property can silently invalidate an existing fixture that
assumed it stayed unmodelled.** Nothing catches this except reading every
fixture a promotion touches; grep for the property's name across `testdata/`
before trusting a green `TestRoundTrip`.

## A shared default was wrong for one caller

TZ and RELATED both carry a VALUE parameter that has to be read explicitly
rather than sniffed from the value -- unlike `tel:`, neither URI form has a
fixed scheme prefix. Both were written against one `valueType` helper that
defaults to text when VALUE is absent, because every other property in this
package that carries VALUE does default to text.

RELATED does not. Section 6.6.6's own examples write a bare URI with no
parameter at all and reserve `VALUE=text` for the free-text form -- the
default is uri. Reusing the shared helper silently routed every unmarked
RELATED value onto the text branch, discarding the URI entirely and losing
no bytes in the process: the wrong branch still held a string, so nothing
crashed or emitted an empty field.

`TestDecodeNewProperties` -- specific-value assertions, the same fix
`decode_test.go` needed the first time (§2.6 in `codec-findings.md`) --
caught it. Round-trip equality would not have: decoding onto the wrong oneof
branch and re-encoding from that branch is stable, just wrong from the first
decode onward.

**A shared default is only safe when every caller agrees with it.** Check
the property's own ABNF before reusing a helper that has one baked in,
rather than assuming the common case generalises.

## RFC 9554 found silent truncation, then a format that cannot carry it

Adding RFC 9554's components to ADR and N turned up two things, neither of
which any linter can see.

**Eleven ADR components were being parsed and thrown away.** `decodeAddress`
split the value on semicolons and then read the first seven, so an
eighteen-component ADR lost everything past the country -- room, floor,
block, landmark, all of it -- with no error. The encoder was blind in exactly
the same way, so decode -> encode -> decode stayed equal while the data was
gone. This is the same shape as the TYPE bug in `codec-findings.md`, and the
same lesson: **round-trip equality proves stability, not correctness.**
`TestRfc9554Components` asserts the components by value for that reason.

**xCard cannot represent them at all, and that is the format's doing.** RFC
6351 maps a compound value to *named* child elements -- `<surname>`,
`<street>` -- so every component needs a registered XML name. RFC 9554
defines its components only for the text syntax and registers none. The
choice was to invent element names or to drop the components crossing into
XML; inventing them would emit xCard no other implementation could read, so
they are dropped and `TestXCardCrossFormat` ignores exactly those fields with
the reason written next to them.

jCard has no such problem. RFC 7095 section 3.3.1.3 writes a compound value
as a positional array, which simply gets longer, so the same data crosses
into JSON untouched and `TestJCardCrossFormat` passed without a change.
**A format's extensibility is a property of how it encodes structure**, and
two encodings of one model can differ in what they are *able* to say.

## The quoted-parameter rule was inverted, and it corrupted names

RFC 9554 section 4.5 adds a LABEL parameter holding a formatted address.
Wiring it turned up that `contentline.Parse` **unquoted a parameter value
before splitting it on commas**, so every comma inside a quoted-string was
treated as a value separator.

`SplitUnescaped` already honoured quoting; the bug was purely the order of
the two steps. The blast radius was every free-text parameter in both
formats, not just LABEL:

```
CN="Doe, John"   ->  "Doe"          # the given name, gone
```

Both RFCs are unambiguous. RFC 5545 section 3.1: *"Property parameter values
that contain the COLON, SEMICOLON, or COMMA character separators MUST be
specified as quoted-string text values."* RFC 6350 section 3.3's `param-value`
grammar says the same. A quoted-string is one value.

**But the RFC contradicts itself, and the codec has to serve both readings.**
RFC 6350 section 6.4.1's own example is `TEL;TYPE="work,voice"` meaning two
types -- which section 3.3's grammar does not permit. Real vCards follow the
example, and `testdata/rfc6350_example.vcf` carries that exact line. So the
parser splits inside quotes for the parameters the RFCs define as lists
(TYPE, PID, SORT-AS, DELEGATED-FROM, DELEGATED-TO, MEMBER) and never for
anything else.

Getting either half wrong is silent, and in opposite directions: split too
eagerly and a name truncates, split too timidly and a TYPE list collapses to
one unrecognised token that then gets dropped. `TestQuotedParamIsOneValue`
asserts both halves in one test so neither can be fixed at the other's
expense.

**A round trip could not have caught this.** The encoder re-quoted whatever
truncated value came back, so decode -> encode -> decode was stable while
half the name was gone -- the third time that exact shape has appeared in
this repository. Symmetric loss is invisible to stability tests by
construction.

With the parser fixed, LABEL is mapped: `Address.label` decodes, re-quotes on
encode via `contentline.EscapeParam`, and round-trips with its comma intact.

## The vCard to JSContact conversion, and where it deliberately disobeys

`codec/go/jscontact` implements RFC 9555. Three things about it are worth
knowing before changing it.

**It ignores a MUST in RFC 9555.** Section 2.1.1 requires an implementation
converting a vCard with no UID to *generate* one, because RFC 9553 made the
Card's `uid` mandatory. RFC 9982 then made `uid` optional for exactly that
reason: a generated identifier differs on every run, so re-importing the same
vCard creates a duplicate rather than matching the record already stored. The
later RFC wins, and `TestUidIsNotGenerated` asserts two conversions of one
vCard agree.

**Two rules exist only to prevent silent duplication**, and both are in Table
1 of section 2.5.5 and the table in section 2.6.1:

- Converting *to* vCard, `surname2` is appended to Family name and
  `generation` to Honorific suffix, so a reader that predates RFC 9554 still
  sees the whole name. Converting *from* vCard, a value found in both the
  merged slot and its dedicated component is taken once.
- The legacy `street address` and `extended address` slots convert **only if
  the ADR is not already in RFC 9554's extended form**, for the same reason.

Drop either and the name or the street doubles on every round trip -- and
doubles *silently*, because the vCard stays valid and the Card still parses.
Both are mutation-tested: removing the dedupe fails `TestNameMergeRule`,
removing the format switch fails `TestAddressFormatSwitch`.

**Some conversions are lossy by construction, and the RFC says so.** TITLE and
ROLE are two vCard properties and one JSContact object keyed on `kind`; IMPP
and SOCIALPROFILE both become `onlineServices`, distinguished on the way back
only by whether a `service` is set. A birthday of "circa 1800" -- which RFC
6350 section 6.2.5 permits -- has no PartialDate form at all and converts to
nothing rather than to a wrong date. **Round tripping preserves the data, not
the encoding**, and a consumer told otherwise will eventually be surprised.
