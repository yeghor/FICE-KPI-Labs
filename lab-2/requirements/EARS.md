# Requirements Specification (EARS)

## 1. Actors
- **Client:** User that takes part in an registered organization
- **Interface** One of the accessible interfaces (WebUI, CLI, RESTful API, etc.)

### Requirements

| REQ    | TYPE|  WHEN | SHALL | 
| ------ | ----| ---- | ----- |
| REQ-01 | Unwanted Behavior | **IF** the Data-stream runner stops to process queues | The system **SHALL** perform safe-exit to allow ArgoCD reboot the pod |
| REQ-02 | Unwanted Behavior  | **IF** dead data-streams are noticed | The system **SHALL** mark the data-streams as degraded and stop processing them until the streams'll be restarted manually or by another worker |
| REQ-03 | Event-Driven | **WHEN** the user tries to retrieve analyze of incidents in a specific date range | The system **SHALL** return readable and visualy appealing representation of `Incidents` of a chosen source |
| REQ-04 | Event-Driven  | **WHEN** the user tries to create records of DNN model metrics | The system **SHALL** create records in corresponding database table with initial status |
| REQ-05 | Event-Driven  | **WHEN** the user tries to fine-tune the DNN model via stored evaluation metrics table | The system **SHALL** start runner to feed new stored training instances with a correct label |
| REQ-06 | Ubiquitous  | | The system **SHALL** restrict external access (from outside the organization) to the data the organization owns|
| REQ-07 | Event-Driven | **WHEN** the user requests the latest incidents | The system **SHALL** return the latest incidents ordered by confidence |

### Datamodel Changes Required

- `VIDEO_STREAMS` - new field `degraded BOOLEAN DEFAULT False` to distinct degraded entries from active one's
- New table - `EVALUATION_METRICS_RECORDS` to store useful incident cases reviewed by a user for DNN fine-tuning; each record has a `status` (e.g. `reviewed`) and a nullable `metric` whose value is one of `FP`, `TP`, `TN`, or `FN`
