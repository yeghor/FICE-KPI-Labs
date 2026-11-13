# Sequence Diagrams

Sequence diagrams for every use case in [Gherkin.md](./Gherkin.md).

**Notation**
- `actor`: a human or external system that starts or watches the flow
- `participant`: a system component or object
- `->>`: synchronous call; `-)`: asynchronous message; `-->>`: return
- Activation bars: `+` / `-` on a message, or `activate` / `deactivate`
- Combined fragments: `opt`, `alt`/`else`, `loop`, `par`/`and`

**Fragment coverage**

| UC | Actors | opt | alt | loop | par |
| :--- | :--- | :---: | :---: | :---: | :---: |
| UC-01.1 | ArgoCD | ✓ | ✓ | ✓ | ✓ |
| UC-01.2 | Client, Recovery Worker | ✓ | ✓ | ✓ | ✓ |
| UC-02.1 | Client | ✓ | ✓ | ✓ | ✓ |
| UC-02.2 | Client | ✓ | ✓ | ✓ | ✓ |
| UC-03.1 | Client | ✓ | ✓ | ✓ | ✓ |
| UC-03.2 | Client | ✓ | ✓ | ✓ | ✓ |
| UC-04.1 | User / Interface | ✓ | ✓ | ✓ | ✓ |

---

## Feature 01: Data-stream health

### UC-01.1: Exit safely when a runner stops (US-01, REQ-01)

```mermaid
sequenceDiagram
    autonumber
    actor Argo as ArgoCD
    participant K8s as Kubernetes (kubelet)
    participant Runner as Data-stream Runner
    participant WD as Queue Watchdog
    participant Queue as Message Queue
    participant DB as Database

    Note over Runner,Queue: Given: the runner is processing queues
    activate Runner
    par Queue processing
        loop every poll interval
            Runner->>+Queue: consume(streamQueues)
            Queue-->>-Runner: batch | timeout | error
            opt batch received
                Runner->>Runner: process frame sequences
                Runner->>Queue: ack(batch)
                Runner-)WD: heartbeat(now)
            end
        end
    and Liveness supervision
        loop every liveness probe period
            K8s->>+WD: GET /healthz
            WD->>WD: age = now - lastHeartbeat
            alt age <= stallTimeout
                WD-->>K8s: 200 OK
            else age > stallTimeout (runner stopped processing)
                WD-->>K8s: 503 Unhealthy
            end
            deactivate WD
        end
    end

    Note over Runner,WD: When: the runner stops processing queues
    WD->>+Runner: requestSafeExit(reason = "stalled")
    Runner->>Runner: stop consuming new messages
    Runner->>+Queue: nack / requeue in-flight messages
    Queue-->>-Runner: requeued
    Runner->>+DB: flush pending incidents, close connections
    DB-->>-Runner: committed
    Runner-->>-WD: shutdown complete
    Runner->>K8s: exit(code = 1)
    deactivate Runner

    Note over Argo,K8s: Then: the pod is rebooted
    activate K8s
    K8s->>K8s: mark container terminated
    K8s--)Argo: application health = Degraded
    deactivate K8s
    Argo->>+K8s: sync desired state (recreate pod)
    K8s->>Runner: start new runner instance
    K8s-->>-Argo: pod Running / Healthy
```

### UC-01.2: Isolate and recover a dead data-stream (US-02, REQ-02)

```mermaid
sequenceDiagram
    autonumber
    actor Client as Client (can_manage_streams)
    actor Worker as Recovery Worker
    participant UI as Interface (WebUI / CLI / REST)
    participant API as Stream API
    participant Mon as Stream Health Monitor
    participant Runner as Data-stream Runner
    participant Src as Video Source
    participant DB as Database (VIDEO_STREAMS)

    Note over Client,DB: New streams default to not degraded
    Client->>+UI: register stream
    UI->>+API: POST /streams
    API->>+DB: INSERT VIDEO_STREAMS (degraded = false by default)
    DB-->>-API: streamId
    API-->>-UI: 201 Created
    UI-->>-Client: stream registered (not degraded)

    Note over Mon,DB: Given: a data-stream is being processed
    loop every health-check interval
        Mon->>+DB: SELECT active streams WHERE degraded = false
        DB-->>-Mon: streams
        loop for each stream
            Mon->>+Src: probe(lastFrameTimestamp)
            Src-->>-Mon: frame age | no response
            alt frames arriving
                Mon->>Mon: reset failure counter
            else no frames for longer than deadTimeout
                Note over Mon,DB: When: the stream is detected as dead
                par persist state
                    Mon->>+DB: UPDATE VIDEO_STREAMS SET degraded = true
                    DB-->>-Mon: ok
                and suspend processing
                    Mon->>+Runner: suspend(streamId)
                    Runner->>Runner: stop consuming the stream queue
                    Runner-->>-Mon: suspended
                end
                opt alerts enabled for the organization
                    Mon-)UI: notify "stream degraded"
                end
            end
        end
    end

    Note over Client,DB: Then: processing resumes after the stream is restarted
    alt manual restart by Client
        Client->>+UI: restart stream
        UI->>+API: POST /streams/{id}/restart
        API->>+Src: restart()
        Src-->>-API: started
        API->>+DB: UPDATE VIDEO_STREAMS SET degraded = false
        DB-->>-API: ok
        API->>+Runner: resume(streamId)
        Runner-->>-API: resumed
        API-->>-UI: 200 OK
        UI-->>-Client: stream active
    else restart by another worker
        Worker->>+Src: restart()
        Src-->>-Worker: started
        Worker->>+DB: UPDATE VIDEO_STREAMS SET degraded = false
        DB-->>-Worker: ok
        Worker->>+Runner: resume(streamId)
        Runner-->>-Worker: resumed
    end
```

---

## Feature 02: Incident review

### UC-02.1: Analyze incidents by source and date range (US-03, REQ-03)

```mermaid
sequenceDiagram
    autonumber
    actor Client
    participant UI as Interface (WebUI / CLI / REST)
    participant API as API Gateway
    participant Auth as Auth Service
    participant An as Incident Analytics Service
    participant DB as Database (INCIDENTS)
    participant Viz as Report Renderer

    Note over Client,UI: Given: a client selects a source and date range
    Client->>+UI: select source (stream) + date range
    UI->>UI: validate range (from <= to)
    alt invalid range
        UI-->>Client: validation error
    else valid range
        Note over Client,API: When: they request an incident analysis
        UI->>+API: GET /incidents/analysis?stream={id}&from&to
        API->>+Auth: authorize(user, can_view_incidents, stream.organization)
        Auth-->>-API: allow | deny
        alt deny
            API-->>UI: 403 Forbidden
            UI-->>Client: access denied
        else allow
            API->>+An: analyze(streamId, from, to)
            par fetch incident list
                An->>+DB: SELECT * FROM INCIDENTS WHERE video_stream_id AND created_at BETWEEN
                DB-->>-An: incidents
            and fetch aggregates
                An->>+DB: SELECT count, avg(confidence) GROUP BY day
                DB-->>-An: daily aggregates
            end
            opt no incidents in range
                An->>An: build empty-state summary
            end
            An->>+Viz: render(incidents, aggregates)
            loop for each incident
                Viz->>Viz: format row (time, confidence, resolved)
            end
            Viz-->>-An: charts + table
            An-->>-API: analysis report
            API-->>UI: 200 OK (report)
            Note over Client,UI: Then: incidents are shown in a readable, clear format
            UI-->>Client: render charts and table
        end
        deactivate API
    end
    deactivate UI
```

### UC-02.2: Review the latest incidents (US-04, REQ-07)

```mermaid
sequenceDiagram
    autonumber
    actor Client
    participant UI as Interface (WebUI / CLI / REST)
    participant API as API Gateway
    participant Auth as Auth Service
    participant Inc as Incident Service
    participant DB as Database

    Note over Client,DB: Given: incidents are available
    Client->>+UI: open "Latest incidents"
    loop auto-refresh every N seconds while the view is open
        Note over Client,API: When: a client requests to view them
        UI->>+API: GET /incidents/latest?limit=N
        API->>+Auth: authorize(user, can_view_incidents)
        Auth-->>-API: allow (orgId) | deny
        alt deny
            API-->>UI: 403 Forbidden
            UI-->>Client: access denied
        else allow
            API->>+Inc: getLatest(orgId, limit)
            par latest incidents
                Inc->>+DB: SELECT latest INCIDENTS of orgId streams
                DB-->>-Inc: incidents
            and stream metadata
                Inc->>+DB: SELECT VIDEO_STREAMS, LOCATIONS of orgId
                DB-->>-Inc: stream and location names
            end
            Inc->>Inc: sort by confidence DESC
            opt no incidents returned
                Inc->>Inc: return empty list
            end
            Inc-->>-API: incidents ordered by confidence
            API-->>UI: 200 OK
            Note over Client,UI: Then: latest incidents ordered by confidence
            UI-->>Client: incident list
        end
        deactivate API
    end
    deactivate UI
```

---

## Feature 03: DNN metrics and fine-tuning

### UC-03.1: Record model metrics (US-05, REQ-04)

```mermaid
sequenceDiagram
    autonumber
    actor Client
    participant UI as Interface (WebUI / CLI / REST)
    participant API as API Gateway
    participant Auth as Auth Service
    participant Met as Metrics Service
    participant DB as Database (EVALUATION_METRICS_RECORDS)
    participant Bus as Event Bus

    Note over Client,UI: Given: a client submits DNN model metrics
    Client->>+UI: submit metrics (incident ids, optional FP / TP / TN / FN)
    UI->>+API: POST /evaluation-metrics
    API->>+Auth: authorize(user, organization)
    Auth-->>-API: allow | deny
    alt deny
        API-->>UI: 403 Forbidden
        UI-->>Client: access denied
    else allow
        API->>+Met: createRecords(payload, orgId)
        loop for each submitted record
            Met->>+DB: incident exists AND belongs to orgId?
            DB-->>-Met: true | false
            Met->>Met: validate metric in {FP, TP, TN, FN} or empty
            opt metric omitted
                Met->>Met: metric = NULL
            end
        end
        alt validation failed
            Met-->>API: validation errors
            API-->>UI: 422 Unprocessable Entity
            UI-->>Client: show errors
        else all records valid
            Note over Met,DB: When: the request is accepted
            Met->>+DB: INSERT records (status = initial)
            DB-->>-Met: record ids
            par respond to client
                Met-->>API: created records
                API-->>UI: 201 Created
                Note over Client,UI: Then: metrics are stored with their initial status
                UI-->>Client: records saved
            and publish event
                Met-)Bus: metrics.recorded(record ids)
            end
        end
        deactivate Met
    end
    deactivate API
    deactivate UI
```

### UC-03.2: Fine-tune from reviewed evaluation metrics (US-06, REQ-05)

```mermaid
sequenceDiagram
    autonumber
    actor Client
    participant UI as Interface (WebUI / CLI / REST)
    participant API as API Gateway
    participant FT as Fine-tuning Service
    participant DB as Database
    participant Orch as Job Orchestrator
    participant Trainer as Training Runner
    participant Reg as Model Registry

    Note over DB: Given: reviewed records with optional FP / TP / TN / FN
    Client->>+UI: request fine-tuning
    UI->>+API: POST /fine-tuning
    API->>+FT: startFineTuning(orgId)
    FT->>+DB: SELECT EVALUATION_METRICS_RECORDS WHERE status = 'reviewed'
    DB-->>-FT: reviewed records
    loop for each reviewed record
        alt metric = TP or FN
            FT->>FT: label = fight
        else metric = TN or FP
            FT->>FT: label = no fight
        else metric is NULL
            FT->>FT: skip (label cannot be determined)
        end
    end
    alt no labelled instances
        FT-->>API: nothing to train
        API-->>UI: 409 Conflict
        UI-->>Client: no eligible instances
    else labelled instances available
        Note over Client,FT: When: a client requests fine-tuning
        FT->>+DB: INSERT training job (status = queued, dataset)
        DB-->>-FT: jobId
        FT->>+Orch: launchRunner(jobId)
        Orch-->>-FT: accepted
        FT-->>API: 202 Accepted (jobId)
        API-->>UI: 202 Accepted
        UI-->>Client: fine-tuning started
    end
    deactivate FT
    deactivate API
    deactivate UI

    opt job was launched
        Note over Orch,Reg: Then: a runner trains only on correctly labelled instances
        Orch-)+Trainer: start(jobId)
        Trainer->>+DB: load labelled instances for jobId
        DB-->>-Trainer: training instances
        par training
            loop for each epoch
                Trainer->>Trainer: train on labelled instances
            end
        and progress reporting
            loop until training finishes
                Trainer-)DB: UPDATE job progress
            end
        end
        Trainer->>+Reg: save fine-tuned model
        Reg-->>-Trainer: model version
        Trainer->>+DB: UPDATE job status = completed
        DB-->>-Trainer: ok
        deactivate Trainer
    end
```

---

## Feature 04: Organization data privacy

### UC-04.1: Restrict access to organization-owned data (US-07, REQ-06)

```mermaid
sequenceDiagram
    autonumber
    actor Req as User / Interface (requester)
    participant GW as API Gateway
    participant Auth as Auth Service
    participant Svc as Resource Service
    participant Pol as Organization Access Policy
    participant DB as Database
    participant Audit as Audit Log

    Note over Req,DB: Given: data belongs to a registered organization
    Req->>+GW: GET /resources/{id} (token)
    GW->>+Auth: validate(token)
    Auth-->>-GW: principal (userId, orgId, role) | invalid
    alt invalid token or unregistered interface
        GW-->>Req: 401 Unauthorized
    else authenticated
        GW->>+Svc: get(resourceId, principal)
        Svc->>+DB: SELECT organization_id of resource
        DB-->>-Svc: owner orgId
        Svc->>+Pol: canAccess(principal, ownerOrgId, permission)
        Pol->>Pol: principal.orgId == ownerOrgId?
        Pol->>Pol: role grants the permission?
        Pol-->>-Svc: allow | deny
        alt deny (requester outside the organization)
            Note over Req,Svc: When: a requester outside the organization asks for the data
            par respond
                Svc-->>GW: AccessDenied
                Note over Req,GW: Then: the system denies access
                GW-->>Req: 403 Forbidden (no data disclosed)
            and audit
                Svc-)Audit: log denied attempt (principal, resourceId)
            end
        else allow (same organization)
            Svc->>+DB: SELECT resource WHERE id AND organization_id = principal.orgId
            DB-->>-Svc: resource
            opt resource embeds related records
                loop for each embedded record
                    Svc->>Svc: drop record if organization_id != principal.orgId
                end
            end
            Svc-->>GW: resource
            GW-->>Req: 200 OK
        end
        deactivate Svc
    end
    deactivate GW
```
