# Affray Notifier Platform Specification

## 1. Purpose

The Affray Notifier Platform monitors live video streams and detects possible
physical fights from sequences of video frames. It records detections as
incidents and makes them available to authorized users for review and
resolution. The platform is intended to alert monitoring clients within seconds
of an incident.

A single frame is not sufficient evidence of violence: actions such as a
high-five or a hug can resemble a fight in an image. Detection must therefore
consider temporal movement and interactions over a rolling sequence of frames,
not classify an incident from an isolated frame.

## 2. Scope

The specification covers:

- Organizations, locations, users, and role-based permissions.
- Registration and activation of video streams.
- Automated detection and recording of incidents associated with streams.
- Incident review, resolution, and notification-related records.

The data model does not define video transport, model training, notification
delivery channels or destinations, or a user-interface design. These are not
specified as implementation requirements here.

## 3. Users and access control

Users belong to an organization and are assigned a role. A role has the
following independent permissions:

| Permission | Intended capability |
|---|---|
| `can_view_streams` | View video stream records or stream status. |
| `can_manage_streams` | Create, update, activate, or deactivate streams. |
| `can_view_incidents` | View detected incidents and their details. |
| `can_resolve_incident` | Mark an incident as resolved and record a comment. |

The platform must enforce these permissions for the corresponding operations.
The data model does not define whether users can have more than one role,
whether permissions are inherited, or whether organization membership alone
limits visibility; those details require confirmation.

## 4. Functional requirements

### 4.1 Organization and location records

- The platform must store organizations and locations, each with a unique
  identifier and name.
- A video stream must be associated with an organization and a location.
- A user must be associated with an organization.
- Organization and location names must be present. Uniqueness requirements are
  not specified by the data model.

### 4.2 Video stream management

- The platform must store a stream's unique identifier, creation timestamp,
  organization, location, name, tags, active status, width, and height.
- A stream must be associated with exactly one organization and one location.
- Users with `can_manage_streams` must be able to manage stream records.
- Only active streams are in scope for ongoing automated monitoring. The model
  does not specify whether inactive streams are retained, paused, or deleted.
- Width and height describe the video dimensions and may be used to compute
  aspect ratio and resolution.

### 4.3 Detection and incident records

- The platform must analyze temporal sequences from monitored streams to
  identify possible fights; implementation may use a suitable sequence-based
  computer-vision model.
- Each recorded incident must include a unique identifier, associated stream,
  creation timestamp, confidence score, and resolved status.
- An incident may include an additional-information URL and a comment; both
  are optional.
- Detection confidence must be represented as a numeric value. The acceptable
  range, threshold for creating an incident, and calibration policy are not
  specified and must be agreed before implementation.
- Newly detected incidents must initially be unresolved.
- Users with `can_view_incidents` may view incidents. Users with
  `can_resolve_incident` may resolve incidents and provide a comment.
- The incident model contains no explicit resolution timestamp, resolving user,
  or incident status beyond the resolved boolean.

### 4.4 Incident notifications

- The model includes incident-recipient records with a notification type and
  an optional latency priority, and incident-alert records with an issue
  timestamp and associated video stream.
- The platform must support a latency priority for each incident recipient.
  When no priority is specified, the recipient must be treated as having the
  highest priority.
- Notification processing must honor recipients' effective priorities when
  scheduling or delivering incident notifications. The priority scale and
  numeric ordering are not defined by the model and must be agreed before
  implementation.
- The platform must preserve these notification-related records if this
  functionality is implemented.
- Delivery behavior, recipient identity or address, supported notification
  types, retry policy, and the relationship between an alert, an incident, and
  its recipients are not sufficiently defined by the model. No delivery
  guarantees are assumed by this specification.

## 5. Data model

| Entity | Attributes represented in the model |
|---|---|
| `ORGANIZATIONS` | `id` (primary key), `name` |
| `LOCATIONS` | `id` (primary key), `name` |
| `USERS` | `id` (primary key), `name`, `role_id` (foreign key), `organization_id` (foreign key) |
| `ROLES` | `id` (primary key), `name`, `can_view_streams`, `can_manage_streams`, `can_view_incidents`, `can_resolve_incident` |
| `VIDEO_STREAMS` | `id` (primary key), `created_at`, `organization_id` (foreign key), `location_id` (foreign key), `name`, `tags`, `active`, `width`, `height` |
| `INCIDENTS` | `id` (primary key), `video_stream_id` (foreign key), `created_at`, `confidence`, optional `additional_info_url`, `resolved`, optional `comment` |
| `INCIDENT_RECIPIENTS` | `id` (primary key), `incident_id` (foreign key), `notification_type`, optional `latency_priority` (integer; defaults to highest priority when unspecified) |
| `INCIDENT_ALERTS` | `id` (primary key), `video_stream_id` (foreign key), `issued_at` |

The model marks identifiers as primary keys and identifies the stated foreign
keys. It does not define nullability beyond the optional incident URL and
comment, uniqueness constraints, deletion behavior, indexes, or concrete
database types and precision.

## 6. Operational qualities

- **Timeliness:** Detection and alerting are intended to happen within seconds
  of an incident. Recipient latency priorities determine notification
  processing order; an unspecified priority is treated as the highest. The
  precise latency target and measurement boundary must be agreed.
- **Detection quality:** The detector must use temporal evidence to reduce
  false positives from benign interactions that resemble violence in a single
  frame. Accuracy targets and evaluation datasets are not specified.
- **Traceability:** Incidents must retain their stream association, creation
  time, confidence, resolution state, and any supplied comment or reference URL.
- **Authorization:** Each operation must be restricted by the relevant role
  permission.

Availability, retention, privacy, encryption, audit logging, and scaling targets
are not specified by the supplied domain overview or data model and require
separate requirements.

## 7. Model questions to resolve

1. **Stream-to-incident cardinality:** `INCIDENTS.video_stream_id` suggests
   each incident belongs to one stream, while the Mermaid relationship is
   drawn as many-to-many. Confirm whether an incident can involve multiple
   streams; then align the relationship and foreign-key design.
2. **Recipient-to-alert linkage:** `INCIDENT_RECIPIENTS` has an
   `incident_id`, while the diagram relates recipients to alerts and
   `INCIDENT_ALERTS` has only a `video_stream_id`. Define how an alert identifies
   its incident and recipients, and whether recipients represent people,
   endpoints, or notification preferences.
3. **Stream recipients:** The diagram also relates streams to incident
   recipients, but the recipient entity has no stream key. Confirm whether this
   relationship is intended and how it should be represented.
4. **Permission assignment:** Confirm whether a user has exactly one role, as
   the singular `role_id` suggests, and whether organization boundaries scope
   access to streams and incidents.
5. **Detection and notification policy:** Define the incident confidence
   threshold, alert timing, notification types and delivery guarantees, and the
   latency-priority scale and its numeric ordering.
6. **Required values and constraints:** Define constraints for names, tags,
   stream dimensions, timestamps, confidence range, and unique records.
