# The Assignment Concept

**Status:** Research / Exploration
**Date:** 2025-02-05

---

## The Scenario

A university manages lab safety through a folder hierarchy:

```
Lab Safety Program (admin: Program Manager)
├── Lab 1 (PI: Alice, members: Bob, Carol)
├── Lab 2 (PI: Dave, members: Eve)
└── Lab 3 (PI: Frank, members: Grace, Heidi)
```

### Action 1: Assign assessments

The Program Manager assigns a **lab safety assessment** to all child labs.
Each lab's PI receives a task: "complete the safety assessment for your lab."

- Alice fills out the assessment for Lab 1
- Dave fills out the assessment for Lab 2
- Frank fills out the assessment for Lab 3

The Program Manager can see: Lab 1 ✅, Lab 2 ✅, Lab 3 (pending)

### Outcome: Acknowledgement tasks (automatic, based on form results)

Alice completes Lab 1's assessment. The form template is configured so that
completing it produces an **outcome**: "require acknowledgement from all lab
members."

The Outcome Service sees this, queries Lab 1's members from the Folder
Service, and creates acknowledgement tasks:

- Bob gets a task: "acknowledge Lab 1's safety assessment"
- Carol gets a task: "acknowledge Lab 1's safety assessment"

Not all assessments produce acknowledgement outcomes. Lab 2 might use a
different template that doesn't require it. The outcome depends on the form
and its results, not a blanket rule.

Alice (the PI) can see: Bob ✅, Carol (pending)

### Outcome: Training requirements (also automatic, from form results)

The assessment for Lab 3 reveals that the lab works with a specific class
of hazardous chemicals. The form template produces a second outcome:
"all members must complete Hazardous Materials Training."

The Outcome Service creates training tasks:

- Grace gets a task: "complete Hazardous Materials Training"
- Heidi gets a task: "complete Hazardous Materials Training"

A different lab's assessment might not trigger this — it depends on what the
PI reported in the form. The outcomes are driven by the data, not blanket
rules.

### The full picture

```
Admin assigns assessments → one task per lab
  Lab Safety Program
  ├── Lab 1 → Alice completes assessment
  ├── Lab 2 → Dave completes assessment
  └── Lab 3 → Frank completes assessment

Form outcomes drive what happens next → zero or more tasks per lab
  Lab 1 outcomes:
    → "require acknowledgement" → Bob acknowledges, Carol acknowledges
  Lab 2 outcomes:
    → (none)
  Lab 3 outcomes:
    → "require acknowledgement" → Grace acknowledges, Heidi acknowledges
    → "require training: Hazardous Materials" → Grace trains, Heidi trains
```

### Tracking

At each level, someone needs to see progress:

| Who              | What they want to know                             |
| ---------------- | -------------------------------------------------- |
| Program Manager  | Which labs have completed their assessment?         |
| Program Manager  | Which labs have full acknowledgement?               |
| Program Manager  | Which labs have members with outstanding training?  |
| Lab PI (Alice)   | Which of my members have acknowledged?              |
| Lab PI (Frank)   | Which of my members have completed training?        |
| Lab member (Bob) | What tasks do I need to complete?                   |

### Five services involved

| Service            | What it knows                                      |
| ------------------ | -------------------------------------------------- |
| **Folder**         | The hierarchy, who belongs to each folder (roles)  |
| **Assignment**     | What needs to be done and by whom                  |
| **Form**           | How to create and render assessment forms           |
| **Outcome**        | What happens as a result of completing a form       |
| **Notification**   | How to reach people (email, in-app, etc.)          |

The central question: **what is the assignment service's role, and what should
a task look like?**

---

## What Is a Task?

A task is a **self-contained record of intent**. It answers three questions:

1. **Who** needs to do something?
2. **What** do they need to do?
3. **Where** does this belong? (organizational context)

A task does NOT perform the action itself. It doesn't create forms, send
emails, or call other services. It's a record — like a sticky note that says
"Alice: complete the safety assessment for Lab 1."

### What a task carries

```
who:   assignedTo (userId, name, email)
what:  action type + reference to the thing (e.g., templateGroupId)
where: folderId (the organizational unit this belongs to)
when:  assignedOn, completedDate
state: ASSIGNED → COMPLETED (or UNASSIGNED)
```

The task has enough information that **any consumer** — a form renderer, a
dashboard, an email template — can read it and know what to do without calling
back to the assignment service.

### What a task does NOT carry

- A pre-created form instance (form creation is the consumer's job)
- Workflow state or orchestration logic
- Knowledge of other services' APIs

---

## The Assignment Service

The assignment service has a narrow responsibility:

1. **Create** a task record
2. **Notify** the assignee
3. **Track** completion status
4. **Query** tasks (by user, by folder, by template, etc.)

That's it. It doesn't know how forms work. It doesn't know what a folder
hierarchy looks like. It doesn't coordinate multi-step workflows.

### The API surface (conceptual)

```
Commands:
  createTask(who, what, where)  → Task
  completeTask(taskId)          → Task
  unassignTask(taskId)          → Task

Queries:
  tasksByUser(userId)           → [Task]
  tasksByFolder(folderId)       → [Task]
  tasksByTemplate(templateId)   → [Task]
```

When `createTask` is called, the service:
1. Stores the record
2. Sends a notification (email, in-app — using the data already in the task)
3. Returns the task

Nothing else. No form creation. No workflow engine. No saga patterns.

---

## Who Does What?

### The fan-out: "assign to all child folders"

When an admin assigns an assessment to all labs, something needs to:
1. Look up the child folders
2. For each folder, look up the members
3. For each member, create a task

**This is the Folder Service's job** (or a coordination layer close to it).
The folder service knows the hierarchy and the role assignments. It iterates
and calls the assignment service once per person.

The assignment service doesn't need to know about folders, children, or roles.
It just receives: "create a task for Alice, for this template, in this
folder."

```
Folder Service:
  children = getChildFolders("Lab Safety Program")
  for each child:
    members = getRoleAssignments(child.id)
    for each member:
      AssignmentService.createTask(member, templateGroupId, child.id)
```

### Form creation: "Alice opens her task"

When Alice clicks on her task, she needs an actual form to fill out. The task
tells her: "complete assessment using template X." Something needs to create
a form instance from that template.

**This is the Form Service's job** (or the UI layer that talks to it). The
task carries the `templateGroupId`. The consumer uses that to create or
retrieve a form.

The assignment service is not involved. It already did its job when it created
the task.

```
Consumer (UI or Form Service):
  task = getTask(taskId)
  form = FormService.createForm(task.templateGroupId, task.assignedTo.userId)
  // user fills out form
  AssignmentService.completeTask(taskId)
```

### Notification: "you have a new task"

When a task is created, the assignee should be notified. The task already has:
- Who to notify (`assignedTo.email`)
- What it's about (`name`, `action type`)
- Organizational context (`folderId`)

The assignment service calls the notification service with this data. No
workflow step needed — it's a direct side effect of task creation.

---

## Task Types

### Known task types

**Complete Form** — "fill out this assessment form."

```
action:    complete-form
reference: { templateGroupId }
who:       assignedTo
           subject? (optional — the person the form is *about*)
where:     folderId
consumer:  Form Service (creates a form instance, renders it)
```

**Acknowledge Form** — "read and agree to this document."

```
action:    acknowledge-form
reference: { formId }
who:       assignedTo
where:     folderId
consumer:  Form Service (shows the existing form, records acknowledgement)
```

**Complete Training** — "complete this training course."

```
action:    complete-training
reference: { courseId }
who:       assignedTo
where:     folderId
consumer:  Training/LMS platform (delivers the course, tracks completion)
```

### The pattern

All three follow the same shape:

```
who:       person assigned to do the thing
what:      action + reference (enough for a consumer to act)
where:     folder (organizational context)
```

The assignment service doesn't need to understand what "complete-form" or
"complete-training" means. It stores the action and reference. The right
consumer picks it up and knows what to do.

### A task is action + reference

The `action` tells you *what kind of thing* to do. The `reference` gives you
*enough data* to do it. Together they're the "what" of the task.

```
action:    "complete-form"
reference: { templateGroupId: "lab-safety-v2" }

action:    "acknowledge-form"
reference: { formId: "form-abc-123" }

action:    "complete-training"
reference: { courseId: "hazmat-101" }
```

New task types don't require changes to the assignment service. You add a new
`action` value and a new `reference` shape. The assignment service stores it
as-is. A new consumer knows how to handle it.

### What the assignment service doesn't know

The assignment service has no idea:
- What a `templateGroupId` is or how to create a form from it
- What a `courseId` is or where the training platform lives
- Whether the consumer is a web app, a mobile app, or a batch job

It just stores: "Bob needs to do `complete-training` with reference
`{ courseId: 'hazmat-101' }` for Lab 3." Whoever handles training tasks
reads that and acts on it.

---

## Independent Actions, Connected by Outcomes

Assessment and acknowledgement are **independent concerns**. Not every form
needs acknowledgement. The admin assigns assessments. Whether acknowledgement
happens depends on the **outcomes** of the completed form.

### The Outcome Service

When a form is completed, the Form Service doesn't just store answers — it
produces **outcomes**. The Outcome Service interprets those results and
decides what happens next.

One possible outcome: "all members of this folder should acknowledge this
form."

```
Alice completes Lab 1's safety assessment
  → Form Service: form submitted, results stored
  → Outcome Service: evaluates form results
     → Outcome: "require acknowledgement from all Lab 1 members"
     → Outcome Service queries Folder Service for Lab 1 members
     → For each member: calls Assignment Service to create acknowledge task
```

The admin made the original decision by choosing a template whose outcomes
include acknowledgement. But the fan-out is **automated through the outcome**,
not a manual second action.

Other possible outcomes from a completed form:
- Require all members to complete a specific training course
- Flag a safety violation for review
- Generate a compliance report
- Nothing — not all forms produce outcomes that create tasks

### What each service knows (and doesn't)

```
Form Service:
  Knows: form was submitted, here are the results
  Doesn't know: what to do with those results

Outcome Service:
  Knows: these results mean "require acknowledgement from folder members"
  Knows: how to call Folder Service to get members
  Knows: how to call Assignment Service to create tasks
  Doesn't know: how forms work internally, how tasks are stored

Assignment Service:
  Knows: create a task, notify the person
  Doesn't know: why this task exists or what triggered it

Folder Service:
  Knows: who belongs to Lab 1
  Doesn't know: about forms, outcomes, or tasks
```

### Both actions still look the same to assignment

From the assignment service's perspective, it doesn't matter whether the
caller is an admin, the outcome service, or a batch job. It's always the
same operation: **create a task for a person**.

```
Whoever calls:
  createTask(who: Bob, what: acknowledge-form + formId, where: Lab 1)
```

The assignment service doesn't know these are "related" to an assessment.
It doesn't know one was triggered by an outcome. Each is just a task with
data.

---

## Tracking Progress

Because tasks are self-contained, tracking is just querying.

### Program Manager view: "Which labs completed the assessment?"

Query: all complete-form tasks for this template, grouped by folder.

```
tasksByTemplate(templateGroupId) → filter type "complete-form"

Result:
  Lab 1 → COMPLETED (Alice, Jan 15)
  Lab 2 → COMPLETED (Dave, Jan 18)
  Lab 3 → ASSIGNED  (Frank, pending)
```

This works because each task has `folderId` and `status`. No joins needed.

### Program Manager view: "Which labs have full acknowledgement?"

Query: all acknowledge-form tasks for this form, grouped by folder. A lab
is "fully acknowledged" when all its acknowledgement tasks are COMPLETED.

```
tasksByFolder(folderId: "lab-1") → filter type "acknowledge-form"

Result:
  Lab 1: Bob ✅, Carol (pending)  → 1/2
```

Note: Lab 2 has no acknowledgement tasks at all — the admin never assigned
them. That's fine. Acknowledgement is optional and independent.

### Lab PI view: "Who in my lab has acknowledged?"

Same query — tasks for this folder, filtered to acknowledgements.

```
tasksByFolder(folderId: "lab-1") → filter type "acknowledge-form"

Result:
  Bob   → COMPLETED (Jan 16)
  Carol → ASSIGNED  (pending)
```

### Lab member view: "What do I need to do?"

Query: all tasks assigned to me.

```
tasksByUser(userId: "bob")

Result:
  "Acknowledge Lab 1 Safety Assessment" → ASSIGNED
```

### What makes this work

All tracking is just **filtering and grouping tasks**. No aggregation
service. No separate tracking table. The task collection IS the tracking
system because each task carries:

- `folderId` — group by lab
- `status` — filter by completion
- `taskType` — distinguish assessments from acknowledgements
- `assignedTo` — filter by person
- `templateGroupId` or `formId` — filter by which assessment/form

---

## Lifecycle

```
                    ┌─────────────┐
                    │  ASSIGNED   │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              │                         │
              ▼                         ▼
     ┌────────────────┐       ┌────────────────┐
     │   COMPLETED    │       │  UNASSIGNED    │
     └────────────────┘       └────────────────┘
```

- **ASSIGNED** — task exists, waiting for the user to act
- **COMPLETED** — user performed the action
- **UNASSIGNED** — task was revoked before completion

The assignment service only tracks these states. It doesn't know or care
*how* the action was performed — just that it was.

---

## The Notification Question

Notifications are a side effect of task creation. But there are nuances:

### What goes in the notification?

The task has everything needed for a basic notification:
- Recipient: `assignedTo.email`
- Subject: `name` (e.g., "Lab Safety Assessment")
- Context: folder name, assigner name

But what about a **link**? For an AcknowledgeFormTask, we have the `formId`
and can link directly to the form. For a CompleteFormTask, there's no form
yet — the link would go to the task itself (or a landing page that creates the
form on first visit).

### Should the assignment service call the notification service directly?

Options:
- **Yes, directly** — task creation always triggers a notification. Simple.
  The assignment service has a dependency on the notification service.
- **Event-based** — task creation emits an event ("task.created"). A separate
  listener sends the notification. Decoupled, but adds infrastructure.
- **Caller decides** — the caller passes a flag or calls a separate endpoint
  to trigger notification. Flexible, but callers need to remember.

The simplest answer: **yes, directly.** A task without a notification is a
task nobody knows about. Notification is inherent to creating a task, not an
optional add-on.

---

## Querying Tasks

With folder context on tasks, useful queries emerge:

| Query                              | Use case                                      |
| ---------------------------------- | --------------------------------------------- |
| `tasksByUser(userId)`              | "What do I need to do?"                       |
| `tasksByFolder(folderId)`          | "What's the status of Lab 1's assessments?"   |
| `tasksByTemplate(templateGroupId)` | "Who has completed this assessment?"           |
| `tasksByAssigner(userId)`          | "What have I assigned?"                        |

These are all simple filters on the task collection. No joins, no cross-service
calls. Because the task is self-contained, all the data needed for these
queries lives in the task itself.

---

## Design Decisions

The following sections resolve the open questions from earlier exploration.
Each takes a position with rationale. Decisions are grouped by concern area
and reference the original question numbers for traceability.

---

## Outcome Service Design

*Resolves questions 1–4.*

### Position: Standalone service, outcome rules configured per template

The Outcome Service is a **standalone service** — not logic embedded in the
Form Service. The Form Service's job ends when it stores answers. What those
answers *mean* is a separate concern. Keeping them apart means:

- Form templates can be reused across contexts without carrying outcome logic
- Outcome rules can change without redeploying or modifying the Form Service
- The Form Service remains a pure data-entry tool

### How outcomes are configured

Outcome rules live on the **form template** (or template group), not on
individual form instances or folders. A template author defines: "when this
form is completed and the answers match these conditions, produce these
outcomes."

```
OutcomeRule:
  templateGroupId:  "lab-safety-assessment-v2"
  condition:        expression evaluated against form answers
  outcomeType:      "require-acknowledgement" | "require-training" | "flag-for-review" | ...
  config:           { ... outcome-specific data }
```

**Why per template, not per folder?** Because the outcome is a property of
*what was reported*, not *where it was reported*. Lab 1 and Lab 3 use the
same template — but Lab 3's answers trigger training because of the chemical
class reported, not because of anything special about Lab 3's folder.

Folder-level overrides are possible but should be the exception. If a campus
wants a stricter policy ("always require training for any lab in this
building"), that's a folder-level rule *layered on top of* the template rules,
not a replacement. This mirrors the folder-concept-v5 property inheritance
pattern: closest scope wins, global rules provide defaults.

### Known outcome types

| Outcome Type              | What it does                                           |
| ------------------------- | ------------------------------------------------------ |
| `require-acknowledgement` | Create acknowledge-form tasks for folder members       |
| `require-training`        | Create complete-training tasks for folder members      |
| `flag-for-review`         | Create a review task for a designated reviewer role     |
| `generate-report`         | Trigger a report generation job (no task created)       |
| `restrict-access`         | Call Folder Service to modify access (no task created)  |

Not all outcomes create tasks. Some trigger side effects in other services.
The Outcome Service is the router — it evaluates rules and dispatches to
the appropriate service.

### Folder context flows through the task

When the original `complete-form` task is created, it carries `folderId`.
When Alice opens the task and the Form Service creates a form instance, the
form stores the `taskId` (or at minimum `folderId`) as metadata. When the
form is submitted, the completion event includes this context:

```
FormCompleted event:
  formId:           "form-abc-123"
  templateGroupId:  "lab-safety-assessment-v2"
  completedBy:      "alice"
  folderId:         "lab-1"          ← carried from the original task
  answers:          { ... }
```

The Outcome Service receives `folderId` as part of this event. It never
needs to guess or look up where the form came from — the context is
**threaded through from task → form → completion event → outcome evaluation**.

This is the simplest approach and avoids a reverse-lookup problem. The form
doesn't need to "know about" folders in a deep way — it just passes metadata
through.

### Multiple outcomes from a single form

**Yes — a single form can produce multiple independent outcomes.** Lab 3's
assessment might produce both "require acknowledgement" and "require
training." These are evaluated and executed **independently**:

```
Outcome evaluation for Lab 3's completed assessment:
  Rule 1: condition matches → require-acknowledgement → create 2 tasks ✅
  Rule 2: condition matches → require-training         → create 2 tasks ✅
  Rule 3: condition fails  → flag-for-review           → skip
```

Each outcome is processed in isolation. If acknowledgement tasks are created
successfully but training task creation fails, the acknowledgement tasks
still stand. The failed outcome is retried independently.

This means the Outcome Service processes outcomes as a **list of independent
effects**, not an all-or-nothing transaction. Partial success is acceptable
because each outcome is independently meaningful — Bob's acknowledgement task
doesn't depend on Grace's training task.

### Outcomes are eventual, not immediate

**Asynchronous processing.** When a form is submitted:

1. The Form Service stores the answers and returns success to the user
2. The Form Service emits a `form.completed` event
3. The Outcome Service picks up the event and evaluates rules
4. For each matching rule, the Outcome Service dispatches the effect

The user sees "form submitted successfully" immediately. Outcome processing
happens in the background. This means:

- A slow outcome (e.g., creating 50 acknowledgement tasks) doesn't block
  the user
- If the Outcome Service is temporarily down, the event is queued and
  processed when it recovers
- The user doesn't see "your form triggered 3 outcomes" in real-time (this
  is fine — outcomes are an admin concern, not the submitter's concern)

**Idempotency is required.** Because events can be delivered more than once
(at-least-once delivery), the Outcome Service must be idempotent. Each
outcome evaluation should check: "have I already processed this form
completion?" A simple approach: store a `processedEventId` for each outcome
execution and skip duplicates.

```
Before creating tasks:
  if outcomeAlreadyProcessed(formId, outcomeRuleId):
    skip (already handled)
  else:
    create tasks
    markOutcomeProcessed(formId, outcomeRuleId)
```

---

## Task Completion Patterns

*Resolves questions 5–7.*

### Position: The consumer marks the task complete — pattern varies by boundary

There's no single answer because the completion boundary differs per task
type. The guiding principle: **whoever knows the action is truly done is
responsible for marking the task complete.**

### Internal consumers (Form Service): server-side callback

For `complete-form` and `acknowledge-form` tasks, the Form Service is the
consumer. When a form is submitted:

1. User submits form in the UI
2. UI calls Form Service: `submitForm(formId, answers)`
3. Form Service stores the answers
4. Form Service calls Assignment Service: `completeTask(taskId)`
5. Form Service emits `form.completed` event (for outcomes)
6. Form Service returns success to the UI

The Form Service makes the `completeTask` call **server-side, as part of
form submission.** This is a direct service-to-service call, not a UI
responsibility. The Form Service already knows the `taskId` because it was
passed when the form was created (or stored as form metadata).

**Why not the UI?** Because the UI calling two services creates a
partial-failure window. If the form submits successfully but the
`completeTask` call fails (network error, timeout), the form is done but the
task still shows as assigned. Server-side is more reliable.

**Why not event-based?** For internal services we control, a direct call is
simpler and provides immediate consistency. Events add infrastructure
overhead for a problem that a direct call solves. The Form Service already
has a dependency path to the Assignment Service — it reads task data to
create forms, so calling `completeTask` is not a new coupling.

```
Form Service (server-side):
  submitForm(formId, answers):
    store answers
    completeTask(task.id)          ← direct call to Assignment Service
    emit "form.completed" event    ← for Outcome Service (async)
    return success
```

### External consumers (LMS): adapter + webhook or polling

For `complete-training` tasks, the consumer is an LMS we may not control.
The pattern depends on what the LMS supports:

**Option A: Webhook from LMS (preferred if available)**

```
LMS → Webhook → Training Adapter → Assignment Service.completeTask()
```

A **Training Adapter** sits between the LMS and the Assignment Service. The
adapter receives LMS webhooks, maps the LMS completion record to the
corresponding task, and calls `completeTask`. The adapter owns the mapping
between LMS course IDs and assignment task IDs.

**Option B: Polling (fallback)**

```
Training Adapter (cron):
  for each ASSIGNED training task:
    check LMS API for completion status
    if completed: completeTask(taskId)
```

Polling introduces latency (minutes to hours) but doesn't require the LMS
to support webhooks. Acceptable for training completions that aren't
time-critical.

**Option C: User self-reports (simplest, lowest confidence)**

The user completes training in the LMS, returns to the assignment UI, and
clicks "I completed this." The system could optionally verify against the LMS
before marking complete.

The recommended approach: **start with Option C for MVP, add Option A/B as
integrations mature.** Self-reporting gets the workflow moving; automated
verification adds confidence later.

### Reconciliation for out-of-sync states

When the task status and consumer state diverge (task says ASSIGNED but form
was actually submitted, or task says COMPLETED but consumer has no record):

**Position: Accept eventual consistency + periodic reconciliation.**

A reconciliation job runs periodically (daily or on-demand) and checks:

```
Reconciliation:
  for each ASSIGNED task older than X days:
    check consumer for completion evidence:
      complete-form:     does a submitted form instance exist for this task?
      acknowledge-form:  does an acknowledgement record exist for this formId + userId?
      complete-training: does the LMS show completion for this courseId + userId?
    if evidence found:
      completeTask(taskId)
      log: "reconciled task {taskId} — was completed but not marked"
```

This is a safety net, not the primary mechanism. The primary mechanism (Form
Service callback or LMS adapter) should work almost all the time.
Reconciliation catches the edge cases: network blips, partial failures,
manual completions outside the normal flow.

Users can also manually mark tasks complete via the UI (with appropriate
permissions). This covers the case where automated reconciliation can't find
evidence but the user knows the work is done.

---

## Task Model: References and Display

*Resolves questions 8–10.*

### Position: Loosely-typed reference with required display fields

### Reference as opaque JSON

The `reference` field is a **JSON object that the Assignment Service stores
without validation.** The Assignment Service doesn't interpret the reference —
it stores it and returns it. Consumers validate and interpret it.

```
reference: { templateGroupId: "lab-safety-v2" }    ← complete-form
reference: { formId: "form-abc-123" }               ← acknowledge-form
reference: { courseId: "hazmat-101" }                ← complete-training
reference: { inspectionId: "insp-789", ... }        ← future: some new type
```

**Why loosely typed?** Because the Assignment Service is intentionally thin.
Adding a new task type should not require a deployment of the Assignment
Service. The caller provides the reference; the consumer interprets it. The
Assignment Service is a pass-through store for this field.

**What about bad data?** The Assignment Service validates structural
requirements (reference must be a non-empty JSON object) but not semantic
ones. If a caller passes `{ templateGroupId: "nonexistent" }`, the task is
created successfully — the error surfaces when the consumer tries to act on
it. This is acceptable because:

- The caller (Outcome Service, admin UI, etc.) is the one with context to
  validate references
- The Assignment Service can't validate references without knowing about
  every consumer's data model
- A task with a bad reference is still a valid record ("Bob was asked to do
  something") — the consumer reports the problem when Bob tries to act

### Template version pinning

**Store both `templateGroupId` and `templateId` in the reference.** The
caller provides the specific version at assignment time. The consumer uses
the pinned version by default.

```
reference: {
  templateGroupId: "lab-safety-assessment",
  templateId:      "lab-safety-assessment-v2.3"
}
```

**Why pin?** Because the admin assigned a specific assessment. If the
template is updated to v2.4 between assignment and completion, the user
should fill out the version they were assigned, not a surprise new version
with different questions. Pinning is a compliance concern — the assessment
assigned on January 15 should be the assessment completed on February 3.

**Consumer behavior:** The Form Service reads the `templateId` from the
reference and creates a form from that specific version. If the version no
longer exists (deleted or archived), the Form Service shows an error and the
task may need to be reassigned with the new template.

**If the caller doesn't provide a version?** The consumer falls back to
latest. This handles the case where version pinning isn't important (e.g.,
a simple acknowledgement form that rarely changes).

### Display fields on the task

**Yes — tasks carry display metadata.** The task includes a `name` and
optional `description` set by the caller at creation time:

```
Task:
  action:      "complete-training"
  reference:   { courseId: "hazmat-101" }
  name:        "Hazardous Materials Training"
  description: "Required for all members of labs handling Class 3 chemicals"
```

**Why not resolve from the reference at display time?** Because:

1. The UI would need to call the consumer service (LMS, Form Service) just
   to display a task name. This creates a runtime dependency for a read-heavy
   operation (task lists).
2. If the consumer is slow or down, the task list breaks.
3. The name might be context-specific: "Required due to chemical handling in
   Lab 3" is richer than the generic course title.

**The tradeoff:** Display data is denormalized. If the course name changes
from "Hazardous Materials Training" to "Chemical Safety Training," existing
tasks still show the old name. This is acceptable — the task represents a
point-in-time assignment. What was assigned doesn't change retroactively.

The `name` field is **required** (the Assignment Service enforces this). The
`description` field is **optional**. Neither replaces the `reference` — the
consumer still uses the reference to do its job.

---

## Fan-Out, Scope, and Membership Changes

*Resolves questions 11–14.*

### Position: BFF coordinates fan-out, assignment service accepts bulk

### Fan-out coordination: the BFF

The **BFF (Backend for Frontend)** — or an API gateway / orchestration layer
— coordinates the fan-out. Not the Folder Service, not the Assignment
Service.

```
BFF (handling admin's "assign to all labs" action):
  children = FolderService.getChildFolders("lab-safety-program")
  tasks = []
  for each child:
    pis = FolderService.getRoleMembers(child.id, "PI")
    for each pi:
      tasks.push({ assignedTo: pi, action: "complete-form", reference: {...}, folderId: child.id })
  AssignmentService.createTasks(tasks)     ← bulk call
```

**Why the BFF?**

- **Not the Folder Service** — the Folder Service knows about hierarchy and
  membership, but calling the Assignment Service isn't its concern. Adding
  assignment logic to the Folder Service couples two independent domains.
  The Folder Service shouldn't know about tasks, templates, or notifications.

- **Not the Assignment Service** — the Assignment Service shouldn't query
  the Folder Service. It would need to know about folder hierarchies, roles,
  and membership — all concerns it currently avoids. "Assign to all children"
  is an orchestration concern, not a storage concern.

- **The BFF** — it's already the place where user intent ("assign to all
  labs") gets translated into service calls. It queries the Folder Service
  for structure, then calls the Assignment Service with concrete tasks. This
  keeps both services thin and focused.

### Bulk creation endpoint

**Yes — the Assignment Service should have a bulk `createTasks` endpoint.**

```
Commands:
  createTask(who, what, where)       → Task
  createTasks([{who, what, where}])  → [Task]     ← batch variant
```

The bulk endpoint:
- Accepts an array of task creation requests
- Validates each independently
- Stores all in a single database transaction (or batch write)
- Triggers notifications asynchronously (not inline — fan-out of 50 tasks
  shouldn't mean 50 synchronous notification calls)
- Returns the list of created tasks (or partial success with errors)

**At what scale does this matter?** Even at the 15-task level (3 labs × 5
people), a single bulk call is better than 15 individual calls. At the
5,000-task level, it's essential. The endpoint should handle up to a
reasonable batch size (e.g., 500 tasks per call) and the BFF can chunk
larger fan-outs.

**Notifications for bulk tasks are queued, not inline.** Task creation
returns immediately. Notifications are dispatched asynchronously — the
Notification Service handles its own delivery pace.

```
AssignmentService.createTasks(tasks):
  validate each task
  batch insert into database
  queue notification events for each task    ← async
  return created tasks
```

### Late-joiners

**Position: Membership-change events trigger task backfill.** When a new
member joins a folder, the system checks for outstanding assignments and
creates tasks as needed.

The mechanism:

1. Folder Service emits a `member.added` event when someone joins a folder
2. A listener (part of the BFF or a dedicated reconciliation service)
   receives the event
3. The listener queries: "are there any active assignment campaigns for this
   folder that this new member should participate in?"
4. If yes, create the missing tasks

```
On member.added(folderId, userId):
  activeCampaigns = getActiveCampaignsForFolder(folderId)
  for each campaign:
    if userDoesNotHaveTask(userId, campaign):
      createTask(userId, campaign.action, campaign.reference, folderId)
```

This requires the concept of "active campaigns" (see Tracking section below)
to know *what* tasks should exist for a folder. Without it, the system has
no way to know that the new member should get an acknowledgement task.

**Alternative: Manual assignment.** For V1, it may be acceptable for the
admin or PI to manually assign tasks to new members. The late-joiner
automation is a V2 enhancement that depends on the campaign concept being
in place.

### Re-assignment

**Position: Unassign + create new task. No mutation of `assignedTo`.**

A task is a record of "we asked Bob to do X." If Bob can no longer do it,
that record doesn't change — Bob was asked, and the task was unassigned.
A new task is created for Carol.

```
unassignTask(taskId: "task-1")
  → task-1: Bob, UNASSIGNED

createTask(who: Carol, ...)
  → task-2: Carol, ASSIGNED
```

**Why not mutate?** Because the task history should be auditable. "We
assigned Bob, then unassigned him, then assigned Carol" is more informative
than "Carol has a task" with no history of Bob's involvement. The audit
trail matters in compliance contexts.

The `unassignTask` operation records *who* unassigned and *when*. The new
task is a separate record. Linking them (optional `replacesTaskId` field)
is possible but not required for the core model.

---

## Tracking, Rollup, and Campaigns

*Resolves questions 15–17.*

### Position: Compute on query for rollup, introduce a lightweight campaign entity

### Full completion as a derived status

**Computed on query.** "Is Lab 1 fully acknowledged?" is answered by:

```
tasks = tasksByFolder("lab-1", type: "acknowledge-form")
total = tasks.length
completed = tasks.filter(t => t.status == "COMPLETED").length
fullyAcknowledged = (completed == total)
```

No separate rollup entity. No materialized status. The task collection IS
the source of truth.

**Why not materialize?** Because the set of tasks can change — new tasks
can be added (late-joiners), tasks can be unassigned. A materialized rollup
would need to be updated on every task state change, which is the same
work as computing it on read, but with the added complexity of keeping the
rollup in sync.

**Performance concern:** For large folders (100+ tasks), this query is still
fast — it's a single-table filter on `folderId` + `taskType` + `status`.
With proper indexes, this is sub-millisecond. No cross-service calls needed
because all the data lives in the task record.

The UI can show progress as a fraction or percentage:

```
Lab 1 Acknowledgement: 1/2 (50%)
Lab 3 Training:        0/2 (0%)
Lab 3 Acknowledgement: 2/2 (100%) ✅
```

### Cross-type progress tracking

**Separate queries per action type, displayed side by side.** The Program
Manager sees a dashboard like:

```
                   Assessment    Acknowledgement    Training
Lab 1              ✅ 1/1        ⏳ 1/2             —
Lab 2              ✅ 1/1        —                  —
Lab 3              ⏳ 0/1        ⏳ 0/2             ⏳ 0/2
```

Each column is an independent query: `tasksByFolder(folderId, type)`.

**Why not a combined "compliance score"?** Because different action types
have different weights and meanings. Lumping them into a single number
("Lab 3 is 40% compliant") obscures which specific obligations are
outstanding. The admin needs to know *what's* missing, not just *how much*.

A combined "all outstanding tasks" view is also useful — "Lab 3 has 5
outstanding tasks" — but it complements the per-type view, doesn't replace
it.

### The campaign entity

**Yes — introduce a lightweight campaign.** A campaign represents a
deliberate act of assignment that may produce cascading tasks. It groups
related tasks for tracking and management.

```
Campaign:
  id:               unique identifier
  name:             "Q1 2025 Lab Safety Assessment"
  initiatedBy:      userId (the admin who started it)
  initiatedOn:      timestamp
  templateGroupId:  "lab-safety-assessment-v2"
  targetFolderId:   "lab-safety-program"    (the parent folder targeted)
  status:           ACTIVE | COMPLETED | CANCELLED
```

Each task created as part of this campaign carries the `campaignId`:

```
Task:
  ...
  campaignId:  "campaign-abc"    ← links back to the campaign
```

This includes both the direct tasks (assessments to PIs) and the cascading
tasks (acknowledgements and training created by outcomes). The Outcome
Service receives the `campaignId` through the same context-threading
mechanism as `folderId` — it's carried from task → form → completion event
→ outcome evaluation → new task creation.

### What the campaign enables

| Query                                          | How                                           |
| ---------------------------------------------- | --------------------------------------------- |
| "What's the completion rate for this rollout?"  | `tasksByCampaign(campaignId)` → compute %     |
| "Remind everyone with outstanding tasks"        | Filter ASSIGNED tasks by campaignId → notify  |
| "Cancel all tasks from this rollout"            | Bulk unassign by campaignId                   |
| "Which campaigns are active for this folder?"   | Used by late-joiner logic                     |
| "History of rollouts for this program"          | List campaigns by targetFolderId              |

### Campaign is not a workflow engine

The campaign is a **grouping mechanism**, not an orchestrator. It doesn't
control the order of task creation, manage dependencies between tasks, or
enforce completion sequences. It's a label that connects related tasks for
querying and bulk operations.

The campaign status is derived:
- **ACTIVE** — at least one task in the campaign is ASSIGNED
- **COMPLETED** — all tasks are COMPLETED (or UNASSIGNED)
- **CANCELLED** — admin explicitly cancelled the campaign (all remaining
  ASSIGNED tasks are bulk-unassigned)

---

## Notification Design

*Resolves questions 18–20.*

### Position: Task links resolve through a URL builder, notifications keyed by action type

### Notification links

The task does **not** store a URL. URLs are environment-specific (staging vs
production), change over time, and are a presentation concern. Instead, the
Notification Service resolves the link from the task's `action` and
`reference` using a **URL builder**.

```
URL Builder (within Notification Service or shared utility):
  buildTaskUrl(task):
    switch task.action:
      "complete-form":
        return `/tasks/${task.id}`
        // Landing page that creates the form on first visit
      "acknowledge-form":
        return `/forms/${task.reference.formId}/acknowledge`
        // Direct link to the form
      "complete-training":
        return LMS.courseUrl(task.reference.courseId)
        // Deep link to the LMS course
      default:
        return `/tasks/${task.id}`
        // Fallback: generic task page
```

For `complete-form` tasks, there's no form instance yet. The link goes to a
**task landing page** that:
1. Shows the task details
2. Creates a form instance from the template (if one doesn't exist yet)
3. Redirects to the form

This is a UI concern — the task landing page is the universal entry point
for any task, regardless of type. It reads the task, determines the action,
and routes the user to the right experience.

### Notification templates by action type

**Notifications are templated by action type.** Each action type maps to a
notification template:

```
Notification Templates:
  complete-form:
    subject: "New assessment assigned: {task.name}"
    body: "{assigner.name} assigned you an assessment for {folder.name}."
    cta: "Start Assessment"

  acknowledge-form:
    subject: "Acknowledgement required: {task.name}"
    body: "Please review and acknowledge {task.name} for {folder.name}."
    cta: "Review & Acknowledge"

  complete-training:
    subject: "Training required: {task.name}"
    body: "You are required to complete {task.name} for {folder.name}."
    cta: "Start Training"
```

The template uses fields from the task (`name`, `action`, `reference`) and
resolved context (`folder.name`, `assigner.name`). The Notification Service
resolves folder names and assigner names from the Folder Service / User
Service as needed — this is a read-time enrichment, not data stored on the
task.

**Why by action type, not caller-provided?** Because notification content
should be consistent. Every "complete-form" notification should look the
same, regardless of whether it was created by an admin, the Outcome Service,
or a batch job. Consistency builds user trust — they learn to recognize
task notifications.

The caller **can** provide additional context via the task's `description`
field, which the template can include. But the caller doesn't control the
notification layout or template selection.

### Reminder notifications

**Yes — reminders are supported, configured at the campaign or template level.**

Reminder configuration lives on the **campaign** (or defaults from the
template):

```
Campaign:
  ...
  reminderPolicy:
    enabled:     true
    interval:    7 days          ← remind every 7 days
    maxReminders: 3             ← stop after 3 reminders
    escalateAfter: 21 days      ← after 3 reminders, notify the assigner
```

**Who owns reminders?**

The **Notification Service** owns reminder scheduling and delivery. The
Assignment Service doesn't run cron jobs or manage reminder state. The flow:

1. When a task is created, the Assignment Service informs the Notification
   Service (as part of the initial notification)
2. The Notification Service schedules reminders based on the campaign's
   reminder policy
3. On each reminder interval, the Notification Service checks: is the task
   still ASSIGNED?
   - Yes → send reminder
   - No (COMPLETED or UNASSIGNED) → cancel remaining reminders
4. After `maxReminders`, optionally escalate to the assigner or admin

```
Notification Service (reminder scheduler):
  on task.created:
    if campaign.reminderPolicy.enabled:
      scheduleReminder(task.id, campaign.reminderPolicy.interval)

  on reminder.due:
    task = AssignmentService.getTask(taskId)
    if task.status == "ASSIGNED":
      sendReminder(task)
      if remindersCount < maxReminders:
        scheduleNextReminder(task.id)
      else if escalateAfter reached:
        notifyAssigner(task, "task overdue")
    else:
      cancelReminders(task.id)
```

**Default policy:** If no campaign-level policy is set, the template can
define a default reminder policy. If neither is set, no reminders are sent.
Reminders are opt-in, not default — some assignments are time-sensitive
and warrant reminders, others are ongoing obligations that don't need nagging.

---

## Task Data Model

The concrete shape of a task record, incorporating all decisions above.

```
Task:
  id:              string (uuid)
  campaignId:      string (uuid, optional — null for ad-hoc tasks)

  # Who
  assignedTo:
    userId:        string
    name:          string
    email:         string
  assignedBy:
    userId:        string
    name:          string

  # What
  action:          string ("complete-form" | "acknowledge-form" | "complete-training" | ...)
  reference:       JSON object (opaque to the assignment service, interpreted by consumer)
  name:            string (required — human-readable display name)
  description:     string (optional — contextual detail)

  # Where
  folderId:        string (organizational unit this task belongs to)

  # When
  assignedOn:      timestamp
  completedOn:     timestamp (null until completed)
  unassignedOn:    timestamp (null unless unassigned)
  unassignedBy:    { userId, name } (null unless unassigned)

  # State
  status:          "ASSIGNED" | "COMPLETED" | "UNASSIGNED"

  # Metadata
  createdAt:       timestamp
  updatedAt:       timestamp
```

### Field notes

**`assignedTo` vs `subject`:** Some task types have a subject — the person
the form is *about* — who differs from the assignee. For example, an ergo
evaluation where the Ergonomist (assignee) evaluates an Employee (subject).
When needed, the subject goes in the `reference`:

```
action:    "complete-form"
reference: {
  templateGroupId: "ergo-evaluation-v1",
  templateId:      "ergo-evaluation-v1.2",
  subjectUserId:   "employee-123",
  subjectName:     "Jane Doe"
}
```

This keeps the Task schema stable — `assignedTo` is always "who does the
work" and any role-specific participants live in the reference.

**`campaignId` is optional.** Ad-hoc tasks (a PI manually assigns a single
acknowledgement to a new member) don't belong to a campaign. The field is
null. Campaign-based queries simply skip these. The `campaignId` is not
required for core task operations — it's a grouping convenience.

**`unassignedBy` and `unassignedOn`:** These fields populate when a task
moves to UNASSIGNED. They provide the audit trail for compliance: who revoked
the task and when. Combined with `assignedBy` and `assignedOn`, the full
history of the task's lifecycle is captured without a separate audit log.

### Indexes

The query patterns from the "Querying Tasks" section drive the indexes:

```
Indexes:
  (assignedTo.userId, status)                     → tasksByUser
  (folderId, action, status)                      → tasksByFolder + type filter
  (reference.templateGroupId, action, status)      → tasksByTemplate (requires JSON index)
  (assignedBy.userId, status)                      → tasksByAssigner
  (campaignId, status)                             → tasksByCampaign
  (folderId, campaignId)                           → late-joiner checks
  (status, assignedOn)                             → reconciliation queries, overdue tasks
```

### Campaign record

```
Campaign:
  id:              string (uuid)
  name:            string ("Q1 2025 Lab Safety Assessment")
  initiatedBy:
    userId:        string
    name:          string
  initiatedOn:     timestamp
  templateGroupId: string (the template that started the campaign)
  targetFolderId:  string (the parent folder targeted)
  status:          "ACTIVE" | "COMPLETED" | "CANCELLED"
  reminderPolicy:
    enabled:       boolean
    intervalDays:  number
    maxReminders:  number
    escalateAfterDays: number (optional)
  createdAt:       timestamp
  updatedAt:       timestamp
```

### Outcome Rule record

```
OutcomeRule:
  id:              string (uuid)
  templateGroupId: string (which template this rule applies to)
  condition:       JSON (expression evaluated against form answers)
  outcomeType:     string ("require-acknowledgement" | "require-training" | ...)
  config:          JSON (outcome-specific configuration)
    # For require-acknowledgement:
    #   { targetRole: "member" }  ← who in the folder gets the task
    # For require-training:
    #   { courseId: "hazmat-101", courseName: "Hazardous Materials Training" }
  enabled:         boolean
  createdAt:       timestamp
  updatedAt:       timestamp
```

---

## End-to-End Event Flow

Walking through the full scenario from the opening section, now with all
design decisions applied.

### Phase 1: Admin initiates campaign

```
Program Manager clicks "Assign Lab Safety Assessment to all labs"

BFF:
  1. Create campaign
     campaign = AssignmentService.createCampaign({
       name: "Q1 2025 Lab Safety Assessment",
       initiatedBy: programManager,
       templateGroupId: "lab-safety-assessment-v2",
       targetFolderId: "lab-safety-program",
       reminderPolicy: { enabled: true, intervalDays: 7, maxReminders: 3 }
     })

  2. Resolve targets
     children = FolderService.getChildFolders("lab-safety-program")
     // → [Lab 1, Lab 2, Lab 3]

  3. Build task list
     tasks = []
     for each child:
       pis = FolderService.getRoleMembers(child.id, "PI")
       for each pi:
         tasks.push({
           campaignId: campaign.id,
           assignedTo: pi,
           assignedBy: programManager,
           action: "complete-form",
           reference: {
             templateGroupId: "lab-safety-assessment-v2",
             templateId: "lab-safety-assessment-v2.3"   ← pinned version
           },
           name: "Lab Safety Assessment",
           folderId: child.id
         })

  4. Create tasks in bulk
     AssignmentService.createTasks(tasks)
     // → 3 tasks created (Alice/Lab1, Dave/Lab2, Frank/Lab3)
     // → 3 notification events queued
```

### Phase 2: PI completes assessment

```
Alice clicks the notification link → /tasks/{task-id}

Task Landing Page (UI):
  1. Read task
     task = AssignmentService.getTask(taskId)
     // action: "complete-form", reference: { templateGroupId, templateId }

  2. Create or retrieve form
     form = FormService.createForm({
       templateId: task.reference.templateId,
       userId: task.assignedTo.userId,
       folderId: task.folderId,
       taskId: task.id,
       campaignId: task.campaignId       ← threaded through
     })

  3. Redirect to form
     → /forms/{form.id}

Alice fills out the form and clicks Submit

Form Service (server-side):
  1. Store answers
  2. Mark task complete
     AssignmentService.completeTask(task.id)
  3. Emit event
     emit "form.completed" {
       formId: form.id,
       templateGroupId: "lab-safety-assessment-v2",
       completedBy: "alice",
       folderId: "lab-1",
       campaignId: "campaign-abc",
       answers: { ... }
     }
  4. Return success to Alice
     // Alice sees "Assessment submitted" immediately
```

### Phase 3: Outcome evaluation (async)

```
Outcome Service receives "form.completed" event

  1. Check idempotency
     if alreadyProcessed(formId): skip

  2. Load rules for this template
     rules = OutcomeRules.findByTemplateGroup("lab-safety-assessment-v2")

  3. Evaluate each rule against the form answers
     Rule 1: "require-acknowledgement"
       condition: answers.requiresAcknowledgement == true
       → Alice checked "yes" → MATCHES

     Rule 2: "require-training"
       condition: answers.chemicalClass in ["3", "4", "5"]
       → Alice reported Class 1 → DOES NOT MATCH

  4. Execute matching outcomes

     Outcome: require-acknowledgement
       members = FolderService.getRoleMembers("lab-1", "member")
       // → [Bob, Carol]
       for each member:
         AssignmentService.createTask({
           campaignId: "campaign-abc",       ← same campaign
           assignedTo: member,
           assignedBy: { system/outcome },
           action: "acknowledge-form",
           reference: { formId: form.id },
           name: "Acknowledge Lab 1 Safety Assessment",
           description: "Review and acknowledge the safety assessment completed by Alice",
           folderId: "lab-1"
         })

  5. Mark outcomes as processed
     markProcessed(formId, rule1.id)
```

### Phase 4: Member acknowledges

```
Bob receives notification → /forms/{formId}/acknowledge

Form Service:
  1. Display the completed form (read-only) with acknowledgement action
  2. Bob clicks "I acknowledge"
  3. Store acknowledgement record
  4. Mark task complete
     AssignmentService.completeTask(bobsTaskId)
  5. Return success

Carol has not yet acknowledged → her task remains ASSIGNED
```

### Phase 5: Tracking queries

```
Program Manager opens dashboard

BFF:
  // Assessment progress
  assessments = AssignmentService.tasksByCampaign("campaign-abc", type: "complete-form")
  // → Lab 1: COMPLETED (Alice), Lab 2: COMPLETED (Dave), Lab 3: ASSIGNED (Frank)

  // Acknowledgement progress (only for labs where outcomes created tasks)
  ackTasks = AssignmentService.tasksByCampaign("campaign-abc", type: "acknowledge-form")
  // → Lab 1: Bob COMPLETED, Carol ASSIGNED (1/2)
  //   Lab 2: (none — no acknowledgement outcome)
  //   Lab 3: (none — assessment not yet completed)

Dashboard renders:
  ┌──────────────────────────────────────────────────────┐
  │ Q1 2025 Lab Safety Assessment                        │
  │                                                      │
  │         Assessment    Acknowledgement    Training     │
  │ Lab 1   ✅ Complete    ⏳ 1/2            —           │
  │ Lab 2   ✅ Complete    —                 —           │
  │ Lab 3   ⏳ Pending     —                 —           │
  └──────────────────────────────────────────────────────┘
```

### Phase 6: Late-joiner (V2)

```
Ivan joins Lab 1 as a new member

Folder Service emits "member.added" { folderId: "lab-1", userId: "ivan" }

Late-Joiner Listener:
  1. Query active campaigns for Lab 1
     campaigns = AssignmentService.activeCampaignsForFolder("lab-1")
     // → campaign-abc is ACTIVE

  2. Check what tasks exist for Lab 1 in this campaign
     // Lab 1 has acknowledge-form tasks (from the outcome)
     // Ivan doesn't have one yet

  3. Create task for Ivan
     AssignmentService.createTask({
       campaignId: "campaign-abc",
       assignedTo: ivan,
       action: "acknowledge-form",
       reference: { formId: "form-abc-123" },
       name: "Acknowledge Lab 1 Safety Assessment",
       folderId: "lab-1"
     })

Dashboard now shows:
  Lab 1 Acknowledgement: ⏳ 1/3 (Bob ✅, Carol pending, Ivan pending)
```

---

## Revised API Surface

Incorporating all design decisions, the full API surface of the Assignment
Service:

```
Commands:
  createCampaign(name, initiatedBy, templateGroupId, targetFolderId, reminderPolicy?)
    → Campaign

  createTask(campaignId?, assignedTo, assignedBy, action, reference, name, description?, folderId)
    → Task

  createTasks([...taskRequests])
    → [Task]                     (bulk variant)

  completeTask(taskId)
    → Task

  unassignTask(taskId, unassignedBy)
    → Task

  cancelCampaign(campaignId, cancelledBy)
    → Campaign                   (bulk-unassigns all ASSIGNED tasks)

Queries:
  getTask(taskId)                                     → Task
  tasksByUser(userId, status?, action?)                → [Task]
  tasksByFolder(folderId, status?, action?)            → [Task]
  tasksByTemplate(templateGroupId, status?, action?)   → [Task]
  tasksByAssigner(userId, status?, action?)             → [Task]
  tasksByCampaign(campaignId, status?, action?)        → [Task]
  activeCampaignsForFolder(folderId)                   → [Campaign]

Events emitted:
  task.created    { taskId, campaignId, action, assignedTo, folderId }
  task.completed  { taskId, campaignId, action, assignedTo, folderId }
  task.unassigned { taskId, campaignId, action, unassignedBy }
```

### What the Assignment Service still doesn't know

Even with campaigns and bulk operations, the Assignment Service remains
intentionally ignorant of:

- What a `templateGroupId` is or how to create a form from it
- What a `courseId` is or where the training platform lives
- How the folder hierarchy is structured or who belongs to which folder
- Why a task exists or what triggered its creation
- What outcomes mean or how form answers are interpreted

It stores tasks. It tracks state. It sends notifications. Everything else
is someone else's problem.

---

## Summary of Design Positions

| # | Question | Position |
|---|----------|----------|
| 1 | What does the Outcome Service look like? | Standalone service; rules configured per template |
| 2 | How does it know the folder context? | Context threaded from task → form → completion event |
| 3 | Can a form produce multiple outcomes? | Yes, evaluated and executed independently |
| 4 | Immediate or eventual outcomes? | Eventual (async) with idempotency |
| 5 | Who marks a task complete? | Consumer does — Form Service server-side callback |
| 6 | Training completion flow? | Adapter (webhook/polling) or user self-report for MVP |
| 7 | Out-of-sync task/consumer state? | Eventual consistency + periodic reconciliation |
| 8 | Reference schema: typed or JSON? | Opaque JSON — assignment service doesn't validate semantics |
| 9 | Template version pinning? | Store both templateGroupId and templateId; pin at assignment |
| 10 | Human-readable display fields? | Yes — `name` (required) and `description` (optional) on task |
| 11 | Who coordinates fan-out? | BFF / orchestration layer — not Folder or Assignment Service |
| 12 | Bulk operations? | Yes — `createTasks` batch endpoint with async notifications |
| 13 | Late-joiners? | Membership-change events trigger backfill (V2); manual for V1 |
| 14 | Re-assignment? | Unassign + create new task; no mutation of assignedTo |
| 15 | Full completion as derived status? | Computed on query — no materialized rollup |
| 16 | Cross-type progress tracking? | Separate queries per action type, displayed side by side |
| 17 | Campaign entity? | Yes — lightweight grouping mechanism, not a workflow engine |
| 18 | Notification links? | URL builder resolves from action + reference; task landing page |
| 19 | Notification customization? | Templated by action type; caller adds context via description |
| 20 | Reminder notifications? | Yes — Notification Service owns scheduling; campaign configures policy |
