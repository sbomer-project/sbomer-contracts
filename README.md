# sbomer-contracts

This repository defines the official Avro schema contracts for the **SBOMer NextGen** event-driven services. These schemas act as the source of truth for all event messages, ensuring data consistency and compatibility between microservices.

## Core Event Flow

This system uses a "command-and-control" pattern, with generations and enhancements orchestrated by a service (for now named the "SBOM Service" but in this document refered to the "Orchestrator"). Understanding this flow is key to understanding the purpose of each schema.

1.  **Intake (`...kafka.request`):** An intake service (a "handler") prepares a batch request from user input or external systems (e.g., via an API or a listener) and publishes a **`RequestsCreated`** event. This event contains the *full list* of generation tasks to be done, each with a unique `generationId` and respective type and identifier.

2.  **Orchestration (`...kafka.orchestration`):** The Orchestrator service consumes `RequestsCreated`. For *each* task, it resolves a full "recipe" (one generator + N enhancers) and:
    * Publishes a **`GenerationCreated`** event to start the base SBOM generation.

3.  **Generation (`...kafka.generator`):** A "Generator" worker service consumes `GenerationCreated`, does the work, and publishes **`GenerationUpdate`** events (e.g., `GENERATING`, `FINISHED`, `FAILED`).

4.  **Enhancement (`...kafka.enhancer`):** An "Enhancer" worker service consumes `EnhancementCreated`, does its work, and publishes **`EnhancementUpdate`** events.

5.  **Lifecycle Management (Back to `...kafka.orchestration`):** The Orchestrator listens for all `...Update` events.
    * When it receives a `GenerationUpdate` with `Status: FINISHED`, it publishes a **`GenerationFinished`** event.
    * It then starts the enhancement chain by publishing the *first* **`EnhancementCreated`** event (which includes a unique `enhancementId`).
    * When it receives an `EnhancementUpdate` with `Status: FINISHED`, it publishes an **`EnhancementFinished`** event and then publishes the *next* `EnhancementCreated` event in the recipe.

6.  **Batch Completion (Back to `...kafka.orchestration`):** Once all generation and enhancement steps for *all* tasks in the original batch are complete, the Orchestrator publishes a single **`RequestsFinished`** event.

7.  **Publishing (`...kafka.publisher`):** One or more "Publisher" services consume `RequestsFinished`. They perform the final actions (e.g., uploading to an analysis tool) and publish a final **`PublishFinished`** event.

8.  **Error Handling (`...kafka.error`):** Any service that encounters an unrecoverable error publishes a **`ProcessingFailed`** event.

---

## Schema Organization (Namespaces)

The schemas are organized into namespaces that map directly to their source directory and, more importantly, their **domain of ownership**. The "owner" is the service responsible for publishing events in that namespace.

* `org.jboss.sbomer.events.kafka.common`
    * **Purpose:** Reusable data models ("nouns") shared by multiple events (e.g., `EnhancerSpec`, `GenerationRequestSpec`).

* `org.jboss.sbomer.events.kafka.request`
    * **Owner:** Intake / Handler Services
    * **Purpose:** Events related to the *creation* of new batch requests.
    * **Schemas:** `RequestsCreated`

* `org.jboss.sbomer.events.kafka.orchestration`
    * **Owner:** Orchestrator Service (SBOM Service)
    * **Purpose:** "Command" events that *start* new work (`GenerationCreated`, `EnhancementCreated`) or "capstone" events that signal *completion* of a major lifecycle (`GenerationFinished`, `EnhancementFinished`, `RequestsFinished`).

* `org.jboss.sbomer.events.kafka.generator`
    * **Owner:** Generator Workers
    * **Purpose:** "Status" events *from* the generator.
    * **Schemas:** `GenerationUpdate`

* `org.jboss.sbomer.events.kafka.enhancer`
    * **Owner:** Enhancer Workers
    * **Purpose:** "Status" events *from* the enhancer.
    * **Schemas:** `EnhancementUpdate`

* `org.jboss.sbomer.events.kafka.publisher`
    * **Owner:** Publisher Workers
    * **Purpose:** "Status" events *from* the publisher.
    * **Schemas:** `PublishFinished`

* `org.jboss.sbomer.events.kafka.error`
    * **Owner:** Any Service
    * **Purpose:** A general-purpose event for reporting unrecoverable errors.
    * **Schemas:** `ProcessingFailed`