# Gherkin Scenarios

Scenarios based on [EARS.md](./EARS.md) and [User-stories.md](./User-stories.md).

## **Feature 01: Data-stream health**

### **Scenario UC-01.1: Exit safely when a runner stops** (US-01, REQ-01)
**Given** a data-stream runner is processing queues<br>
**When** it stops processing them<br>
**Then** the system exits safely so ArgoCD can reboot the pod

### **Scenario UC-01.2: Isolate and recover a dead data-stream** (US-02, REQ-02)
**Given** a data-stream is being processed<br>
**When** it is detected as dead<br>
**Then** the system marks it as degraded and suspends processing until restart<br>
**And** processing resumes after the stream is restarted<br>
**And** new streams default to not degraded

## **Feature 02: Incident review**

### **Scenario UC-02.1: Analyze incidents by source and date range** (US-03, REQ-03)
**Given** a client selects a source and date range<br>
**When** they request an incident analysis<br>
**Then** the system presents matching incidents in a readable, clear format

### **Scenario UC-02.2: Review the latest incidents** (US-04, REQ-07)
**Given** incidents are available<br>
**When** a client requests to view them<br>
**Then** the system presents the latest incidents ordered by confidence

## **Feature 03: DNN metrics and fine-tuning**

### **Scenario UC-03.1: Record model metrics** (US-05, REQ-04)
**Given** a client submits DNN model metrics<br>
**When** the request is accepted<br>
**Then** the system stores them with their initial status

### **Scenario UC-03.2: Fine-tune from reviewed evaluation metrics** (US-06, REQ-05)
**Given** reviewed evaluation records are stored with an optional `FP`, `TP`, `TN`, or `FN` metric<br>
**When** a client requests fine-tuning<br>
**Then** the system starts a runner that trains only on instances with a determinable, correct label
<!-- 
## **Feature: Organization data privacy** -->

<!-- ### **Scenario: Restrict access to organization-owned data** (US-07, REQ-06)
**Given** data belongs to a registered organization<br>
**When** a user or interface outside that organization requests it<br>
**Then** the system denies access -->
