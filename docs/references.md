# References

Where to look things up. Every URL here is checked by CI; if one breaks, fix
it here rather than working around it.

Two are expected to 404 for an anonymous fetch and are exempt: the repository
and the BSR module are private, so they resolve only when signed in.

## This project

| What | Where |
|---|---|
| Repository | https://github.com/the-protobuf-project/protobuf-rfc |
| BSR module | https://buf.build/the-protobuf-project/protobuf-rfc |
| api-linter action | https://github.com/the-protobuf-project/setup-google-api-linter |

## API design — the authority when rules conflict

| What | Where |
|---|---|
| AIP index | https://google.aip.dev/ |
| A specific AIP | `google.aip.dev/NNN` — e.g. https://google.aip.dev/131 |
| api-linter rule docs | https://linter.aip.dev/ |
| A specific rule | `linter.aip.dev/AIP/rule-name` — e.g. https://linter.aip.dev/215/foreign-type-reference |

Read the rule page before assuming a linter complaint is wrong. Several
"surprises" in this repo — AIP-140 banning prepositions, AIP-142 reading a
bare `seconds` as a timestamp, AIP-216 reserving both `state` and `status` —
are documented plainly there.

The AIPs that shaped this schema: 122 (naming), 131-135 (standard methods),
140/141/142 (field names and types), 148 (uid), 151 (long-running), 154
(etag), 158 (pagination), 160 (filtering), 163 (validate_only), 164 (soft
delete), 191 (file layout), 192 (comments), 203 (field behavior), 215
(cross-package references), 216 (state).

## Shared types — check here before defining a value type

`google.*` is the only exemption to the cross-package ban (rule 4), so these
are the one kind of type that can be shared rather than copied per package.

| What | Where |
|---|---|
| `google/type` | https://github.com/googleapis/googleapis/tree/master/google/type |
| `google/api` | https://github.com/googleapis/googleapis/tree/master/google/api |
| `google/rpc` | https://github.com/googleapis/googleapis/tree/master/google/rpc |
| `google/longrunning` | https://github.com/googleapis/googleapis/tree/master/google/longrunning |
| Whole repository | https://github.com/googleapis/googleapis |

In use: `type.DayOfWeek`, `type.Month`, `type.TimeZone`, `rpc.Status`,
`longrunning.Operation`, and the `api.*` annotations. Read locally instead of
from the web with:

```sh
buf export buf.build/googleapis/googleapis -o /tmp/gapi
ls /tmp/gapi/google/type
```

Worth knowing about and not yet used: `api/httpbody.proto` (arbitrary content
with a media type, an alternative shape for interchange payloads),
`api/launch_stage.proto` (marking a package alpha/beta), `rpc/code.proto`
(the canonical error codes the service comments name), `type/decimal.proto`,
`type/interval.proto`, `type/localized_text.proto`.

Adopt only where the model matches — see `plan/07-adding-data.md` for the two
that were rejected and why.

## Tooling

| What | Where |
|---|---|
| buf lint rules | https://buf.build/docs/lint/rules/ |
| buf breaking rules | https://buf.build/docs/breaking/rules/ |
| buf managed mode | https://buf.build/docs/generate/managed-mode/ |
| protovalidate | https://buf.build/docs/protovalidate/ |
| protovalidate standard rules | https://buf.build/docs/protovalidate/schemas/standard-rules/ |
| CEL language | https://github.com/google/cel-spec/blob/master/doc/langdef.md |
| protobuf language guide | https://protobuf.dev/programming-guides/proto3/ |
| protobuf style guide | https://protobuf.dev/programming-guides/style/ |

## The RFCs this schema models

Link to a section, never a document: append `#section-<X.Y.Z>` (rule 11).

| RFC | What | Where |
|---|---|---|
| 6350 | vCard | https://www.rfc-editor.org/rfc/rfc6350.html |
| 6351 | xCard, the XML encoding | https://www.rfc-editor.org/rfc/rfc6351.html |
| 7095 | jCard, the JSON encoding | https://www.rfc-editor.org/rfc/rfc7095.html |
| 6352 | CardDAV | https://www.rfc-editor.org/rfc/rfc6352.html |
| 5545 | iCalendar | https://www.rfc-editor.org/rfc/rfc5545.html |
| 5546 | iTIP scheduling | https://www.rfc-editor.org/rfc/rfc5546.html |
| 7265 | jCal, the JSON encoding | https://www.rfc-editor.org/rfc/rfc7265.html |
| 7986 | New iCalendar properties | https://www.rfc-editor.org/rfc/rfc7986.html |
| 4791 | CalDAV | https://www.rfc-editor.org/rfc/rfc4791.html |
| 4519 | LDAP schema | https://www.rfc-editor.org/rfc/rfc4519.html |
| 3986 | URI syntax | https://www.rfc-editor.org/rfc/rfc3986.html |
| 3966 | the `tel` URI | https://www.rfc-editor.org/rfc/rfc3966.html |
| 5870 | the `geo` URI | https://www.rfc-editor.org/rfc/rfc5870.html |
| 6838 | Media types | https://www.rfc-editor.org/rfc/rfc6838.html |
| 9562 | UUIDs | https://www.rfc-editor.org/rfc/rfc9562.html |

Any RFC follows the pattern `rfc-editor.org/rfc/rfcNNNN.html`, with `NNNN`
the number. The `.txt` form is easier to grep for a section listing; the
`.html` form is what a link should point at, because it has the anchors.

**Verify the section number before citing it.** RFC 5545 orders §3.8.2
alphabetically by descriptive name, so DTSTART is §3.8.2.4 and not §3.8.2.3 —
guessing from memory has been wrong here more than once.

## Licensing

| What | Where |
|---|---|
| Apache 2.0 | https://www.apache.org/licenses/LICENSE-2.0.txt |
| IETF Trust legal provisions (BCP 78) | https://trustee.ietf.org/license-info |
