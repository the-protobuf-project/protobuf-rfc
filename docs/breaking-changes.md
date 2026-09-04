# Breaking changes

Every change here was taken while the BSR module had no consumers, so each was
an edit rather than a v2. That window is the whole justification; once a
consumer exists this file stops growing and the next one costs a major
version.

Newest first.

## The legacy layer was renamed for rule 14

`User` became `Vcard` on `vcards/{vcard}`, its package moving from
`protobuf.rfc6350.user.v1` to `protobuf.rfc6350.vcard.v1`. `Group` became
`LdapGroup` on `ldapGroups/{ldap_group}`, `group.proto` becoming
`ldap_group.proto` with its directory unchanged. `Membership` kept its name and
took the new parent segment,
`ldapGroups/{ldap_group}/memberships/{membership}`.

This frees the plain `User` and `Group` names for RFC 7643 SCIM, which is the
canonical identity model under rule 14 ([`ontology.md`](ontology.md)). The
legacy resources keep their full CRUD services; they are not deprecated.

Also renamed with them: the services (`Users` → `Vcards`, `Groups` →
`LdapGroups`), every request message, the `google.api.http` paths, the request
resource fields (`user` → `vcard`, `group` → `ldap_group`) and their id fields,
and the `protobufrfc.dev/User` and `/Group` example strings in
`shared/interchange`'s comments.

**`buf breaking` reports 19 findings against this**, all file and message
deletions, so the PR-only `BreakingChange` job cannot pass on it. That was
accepted rather than worked around — the breaking config stays `use: [FILE]`
with nothing excepted, per rule 1's spirit. Land it as a push to `main`, where
the job does not run, or take the red check deliberately.

## Calendar times are `CalendarTime`, not `Timestamp`

A `Timestamp` can only express RFC 5545 §3.3.5's FORM #2 (absolute UTC), so
floating time, TZID-referenced time and all-day DATE values could not be
represented — a recurring 09:00 standup shifted an hour across a DST boundary.
`CalendarTime` wraps `google.type.Date` and `google.type.DateTime`, which
covers all four cases. Fields renamed with the types: `start`, `end`, `due`,
`completed`, `until`.

## `ExpandEventRequest` takes one `google.type.Interval`

Instead of two loose timestamps.

## Fields graduated out of `extensions`

Additive rather than breaking, recorded here because they moved with the
above: `Contact.birthday` and `Contact.anniversary` (a `DateOrText`, since RFC
6350 §6.2.5 permits a text value like "circa 1800"), and `Event.position` (a
`google.type.LatLng`).
