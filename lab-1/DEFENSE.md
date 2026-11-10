# Defense: Incident Data Model

## Top three audit findings

1. **Stream and incident cardinality**
   - **What was wrong:** `INCIDENTS.video_stream_id` identified one stream, but
     the diagram allowed an incident to relate to multiple streams.
   - **Specification criterion clarified:** §4.3 and §7 now state that each
     incident belongs to exactly one stream and a stream may have zero or many
     incidents.
   - **Model change:** The diagram now shows `VIDEO_STREAMS ||--o{ INCIDENTS`,
     matching the incident's single stream foreign key.
   - **Fix commit:** Not available; no commit was created, as requested.

2. **Incident, resolver, and alert-recipient relationships**
   - **What was wrong:** The resolver foreign key had no diagram relationship;
     recipient records lacked a user identity; alerts identified only a
     stream, leaving their incident association ambiguous. The recipient-alert
     many-to-many link had no defined semantics or delivery-record entity.
   - **Specification criterion clarified:** §§4.3–4.4 and §7 define the
     resolver as one authorized user per resolved incident, recipients as
     users linked to incidents, and alerts as incident-level events. Each
     relationship's cardinality is stated there. Each alert is associated with
     all recipient records of the incident, making alert-to-recipient
     conceptually M:N through the incident. That association is derived, not
     stored as a direct relationship, because individual deliveries are not
     modeled.
   - **Model change:** The diagram adds `INCIDENTS`-to-`USERS`,
     `INCIDENT_RECIPIENTS`-to-`USERS`, and `INCIDENT_RECIPIENTS`-to-`INCIDENTS`
     relationships; `INCIDENT_ALERTS` now references `incident_id`. Its stream
     is determined through the incident, and its recipients are derived from
     the incident's recipient records.
   - **Fix commit:** Not available; no commit was created, as requested.

3. **Unsupported stream-to-recipient relationship**
   - **What was wrong:** The diagram linked streams directly to recipients,
     although recipient records had no stream key and were already incident
     specific.
   - **Specification criterion clarified:** §§4.4 and 7 define each recipient
     as a user selected for one incident; its stream is identified through
     that incident.
   - **Model change:** The unsupported stream-to-recipient link is removed.
     Recipients relate to incidents and users, and the stream is reached
     through the incident.
   - **Fix commit:** Not available; no commit was created, as requested.

## Model structure

Organizations own video streams and users; locations are assigned to streams,
and roles are assigned to users. A stream generates incidents. An incident may
have a resolving user, recipient records, and alert events. Recipient records
identify users selected for that incident. Alert events belong to incidents;
the associated stream can be obtained through the incident. Notification
destinations and per-recipient delivery attempts are outside this model. Each
alert is associated with all recipients of its incident, a
conceptual M:N association derived through the incident rather than stored as a
direct relationship.

## Normalization check

The model separates organizations, locations, users, roles, streams, incidents,
recipients, and alerts into entities with their own identifiers. Attributes
describe their entity, while foreign keys represent relationships. In
particular, `incident_id` is stored on each recipient and alert record, and
`user_id` on each recipient record; stream identity for an alert is obtained
through its incident rather than duplicated on the alert. This avoids
duplicating the stream-to-incident association and avoids storing recipient
lists in a single field. The model is consistent with the usual 1NF/2NF/3NF
checks for the attributes shown; candidate-key, uniqueness, and further
constraint choices remain unspecified in the specification.
