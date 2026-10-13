# User Stories

These stories are derived from the requirements in [EARS.md](./EARS.md). Each story includes acceptance criteria to make the expected behavior verifiable.

## US-01 Recover when a data-stream runner stops

**Related requirement:** REQ-01

**As a** platform operator,  
**I want** the system to exit safely when a data-stream runner stops processing its queues,  
**so that** ArgoCD can restart the pod without leaving the system in an unsafe state.

**Acceptance criteria**

- **Given** a data-stream runner is processing queues, **when** it stops processing them, **then** the system performs a safe exit.
- **Given** the system exits safely, **when** ArgoCD observes the stopped pod, **then** ArgoCD can reboot the pod.

## US-02 Mark dead data-streams as degraded

**Related requirement:** REQ-02; datamodel change: `VIDEO_STREAMS.degraded`

**As a** data-stream operator,  
**I want** streams that are detected as dead to be marked as degraded and excluded from processing,  
**so that** failed streams do not continue to be processed until they are restarted.

**Acceptance criteria**

- **Given** a data-stream is detected as dead, **when** the system handles it, **then** it marks the stream as degraded.
- **Given** a stream is marked as degraded, **when** workers select streams to process, **then** they do not process that stream.
- **Given** a degraded stream has not been restarted, **when** processing is attempted, **then** it remains excluded from processing.
- **Given** the stream is restarted manually or by another worker, **when** it is available again, **then** it can resume processing.
- **Given** a new `VIDEO_STREAMS` record is created, **when** no degraded value is specified, **then** its `degraded` value defaults to `False`.

## US-03 Analyze incidents for a source and date range

**Related requirement:** REQ-03

**As a** client who belongs to a registered organization,  
**I want** to view an analysis of incidents for a chosen source and date range,  
**so that** I can understand the incidents relevant to that source and period.

**Acceptance criteria**

- **Given** I choose a source and a date range, **when** I request an incident analysis, **then** the system returns incidents matching both.
- **Given** matching incidents are returned, **when** they are presented, **then** the representation is readable and visually clear.

## US-04 View the latest incidents by confidence

**Related requirement:** REQ-04

**As a** client who belongs to a registered organization,  
**I want** to view the latest incidents ordered by confidence,  
**so that** I can quickly review the most relevant incident results.

**Acceptance criteria**

- **Given** incidents are available, **when** I request to view incidents, **then** the system returns the latest incidents.
- **Given** the incidents are returned, **when** they are displayed, **then** they are ordered by confidence and presented readably.

## US-05 Record DNN model metrics

**Related requirement:** REQ-05

**As a** client who submits DNN model metrics,  
**I want** the system to create a metrics record with its initial status,  
**so that** the metric can be tracked through its lifecycle.

**Acceptance criteria**

- **Given** I submit a request to create a DNN model metrics record, **when** the request is accepted, **then** the system creates a record in the corresponding database table.
- **Given** the record is created, **when** its status is inspected, **then** it has the initial status.

## US-06 Record reviewed evaluations for DNN fine-tuning

**Related requirement:** REQ-06; datamodel change: `EVALUATION_METRICS_RECORDS`

**As a** client who reviews incident evaluation metrics,  
**I want** each evaluation record to capture its review status and optional confusion-matrix metric,  
**Given the runner processes stored training instances, when it feeds an instance to the model, then the instance has the correct label.** reviewed incident cases can be used as correctly labeled training instances.

**Acceptance criteria**

- **Given** a reviewed incident case is stored in `EVALUATION_METRICS_RECORDS`, **when** its record is created, **then** it has a `status` field (for example, `reviewed`).
- **Given** an evaluation record has a `metric` value, **when** that value is stored, **then** it is one of `FP`, `TP`, `TN`, or `FN`.
- **Given** an evaluation record has no applicable confusion-matrix metric, **when** it is stored, **then** its `metric` field may be null.
- **Given** I request fine-tuning using stored evaluation metrics, **when** the system processes the request, **then** it starts a runner.
- **Given** a record’s metric is null, **when** the runner selects training instances, **then** it excludes that record unless its label can be determined from another source.

## US-07 Keep organization data private

**Related requirement:** REQ-07

**As a** client who belongs to a registered organization,  
**I want** access to organization-owned data to be restricted to users and interfaces within that organization,  
**so that** people outside the organization cannot access its data.

**Acceptance criteria**

- **Given** data belongs to an organization, **when** a user or interface outside that organization attempts to access it, **then** the system denies access.
- **Given** a client accesses data owned by their organization through an available interface, **when** they are authorized, **then** the system applies the organization's access restrictions.