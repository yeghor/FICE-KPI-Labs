# Lab 3 - Sequence Specification

This document describes the participants and interactions for every scenario in
`Gherkin.md`. The system is not an actor. The following external and internal
participants correspond to users and entities used by the sequence diagrams.

| Participant | Purpose |
|---|---|
| Client / User | Starts a scenario through an interface. |
| Interface | WebUI, CLI, or REST interface used to send requests and show results. |
| API Gateway | Receives requests and routes them to services. |
| Auth Service | Checks authentication and organization permissions. |
| Database | Stores organizations, users, roles, streams, incidents, and evaluation metrics. |
| Video Stream | Stream source that provides frames or can be restarted. |
| Runner / Worker | Processes stream frames or fine-tuning jobs. |

## UC-01.1 - Exit safely when a runner stops

**Participants:** ArgoCD, Kubernetes, Queue Watchdog, Data-stream Runner,
Message Queue, Database.

1. ArgoCD deploys a pod containing a Data-stream Runner.
2. The Runner consumes stream queues and sends heartbeats to the Queue
   Watchdog.
3. Kubernetes periodically checks `/healthz`.
4. If the heartbeat exceeds the stall timeout, the Watchdog asks the Runner to
   stop safely.
5. The Runner stops consumption, requeues in-flight messages, and flushes
   pending incident data.
6. The Runner exits with an error code.
7. Kubernetes marks the pod unhealthy and ArgoCD recreates it.

## UC-01.2 - Isolate and recover a dead data-stream

**Participants:** Client, Recovery Worker, Interface, Stream API, Stream Health
Monitor, Data-stream Runner, Video Source, Database.

1. The Client registers a stream with `degraded = false`.
2. The Stream Health Monitor periodically checks active streams.
3. If no frames arrive before the dead timeout, the Monitor marks the stream as
   degraded and stops the Runner from consuming it.
4. The Client or another Worker restarts the Video Source.
5. The database sets `degraded = false`.
6. The Runner resumes processing the stream.

**Alternative:** An already degraded stream or a stream that has frames
arriving remains active and is not marked degraded.

## UC-02.1 - Analyze incidents by source and date range

**Participants:** Client, Interface, API Gateway, Auth Service, Incident
Analytics Service, Database, Report Renderer.

1. The Client selects a video stream and date range.
2. The Interface validates `from <= to`.
3. The API Gateway requests authorization from the Auth Service.
4. If authorized, the Analytics Service queries matching `INCIDENTS` and daily
   aggregates.
5. The Report Renderer creates charts and a table.
6. The Interface shows the report to the Client.

**Alternative: invalid date range**

The Interface rejects `from > to`, shows a validation error, and does not query
the database.

**Alternative: unauthorized access**

The Auth Service denies the request and the API Gateway returns `403 Forbidden`
without incident data.

## UC-02.2 - Review the latest incidents

**Participants:** Client, Interface, API Gateway, Auth Service, Incident
Service, Database.

1. The Client opens the latest-incidents view.
2. The Interface requests incidents periodically.
3. The API Gateway asks the Auth Service to verify `can_view_incidents`.
4. If authorized, the Incident Service queries the latest incidents and related
   stream metadata.
5. The Incident Service orders incidents by confidence from highest to lowest.
6. The Interface shows the incident list.

**Alternative: unauthorized access**

The API Gateway returns `403 Forbidden` and the Interface shows an access-denied
message.

**Alternative: no incidents**

The Incident Service returns an empty list.

## UC-03.1 - Record model metrics

**Participants:** Client, Interface, API Gateway, Auth Service, Metrics
Service, Database, Event Bus.

1. The Client submits incident IDs and optional `FP`, `TP`, `TN`, or `FN`
   metrics.
2. The Interface sends the records to the API Gateway.
3. The Auth Service verifies that the Client belongs to the organization.
4. The Metrics Service verifies that each incident belongs to the organization
   and validates the metric value.
5. If valid, the Metrics Service creates records in
   `EVALUATION_METRICS_RECORDS` with their initial status.
6. The API Gateway returns `201 Created`.
7. The Event Bus receives a metrics-recorded event.

**Alternative: access denied or invalid records**

The system returns `403 Forbidden` or `422 Unprocessable Entity` and does not
create metrics records.

## UC-03.2 - Fine-tune from reviewed evaluation metrics

**Participants:** Client, Interface, API Gateway, Fine-tuning Service,
Database, Job Orchestrator, Training Runner, Model Registry.

1. The Client requests fine-tuning.
2. The API Gateway passes the request to the Fine-tuning Service.
3. The Fine-tuning Service selects reviewed
   `EVALUATION_METRICS_RECORDS`.
4. It maps `TP` and `FN` to the fight label and `TN` and `FP` to the no-fight
   label.
5. Records with no metric are skipped.
6. If labelled records exist, the Service creates a queued training job and
   starts a Training Runner.
7. The Runner trains on the labelled records and saves the model in the
   Model Registry.

**Alternative: no determinable label**

The Fine-tuning Service returns `409 Conflict` and no Training Runner starts.

## UC-04.1 - Restrict access to organization-owned data

**Participants:** Requester, API Gateway, Auth Service, Resource Service,
Organization Access Policy, Database, Audit Log.

1. The Requester asks the API Gateway for a resource.
2. The Auth Service validates the token and returns the principal's
   organization and role.
3. The Resource Service finds the resource owner organization.
4. The Organization Access Policy checks organization membership and permission.
5. If denied, the API Gateway returns `403 Forbidden` and logs the attempt.
6. If allowed, the Resource Service loads the resource and removes embedded
   records from other organizations.
7. The API Gateway returns the allowed resource.

**Alternative: invalid token**

The API Gateway returns `401 Unauthorized` and discloses no resource data.
