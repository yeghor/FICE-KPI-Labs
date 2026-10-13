## Traceability Matrix

| EARS | User Stories | Gherkin (UC) | Note / Status |
| :--- | :--- | :--- | :--- |
| REQ-01 | US-01 | UC-01.1 | **1:1 Connectivity.** Covered. |
| REQ-02 | US-02 | UC-01.2 | **Many-to-One.** One complex Gherkin scenario covers multiple acceptance criteria for stream isolation and recovery. |
| REQ-03, REQ-04 | US-03, US-04 | UC-02.1, UC-02.2 | **Many-to-Many.** The Incident Review feature covers two combined logics of retrieving, sorting, and displaying `Incidents`. |
| REQ-05, REQ-06 | US-05, US-06 | UC-03.1, UC-03.2 | **Many-to-Many.** DNN metrics recording and model fine-tuning share the same underlying data lifecycle and are tested together. |
| REQ-07 | US-07 | - | **Not Covered.** Organization data privacy and tenant isolation are enforced at the infrastructure/middleware level, falling outside the scope of BDD user scenarios. |