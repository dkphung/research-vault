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

## Open Questions

### Outcome Service

**1. What does the Outcome Service look like?**

The Outcome Service is the bridge between "a form was completed" and "things
need to happen." It interprets form results and triggers actions.

- Is it a standalone service, or logic within the Form Service?
- How are outcomes configured? Per template? Per form instance? Per folder?
- What outcomes exist beyond "require acknowledgement" and "require training"?
  (e.g., "flag for review", "generate report", "restrict lab access")
- Does it need to be idempotent? (What if the same form completion is
  processed twice?)

**2. How does the Outcome Service know the folder context?**

When Alice completes Lab 1's assessment, the Outcome Service needs to know
"this belongs to Lab 1" so it can query Lab 1's members. But the form itself
might not store `folderId`. Does:

- The original task carry the folder context, and is it passed through when
  the form is created?
- The form store metadata about where it came from?
- The Outcome Service receive the folder context as part of the completion
  event?

**3. Can a single form produce multiple different outcome types?**

Lab 3's assessment might produce both "require acknowledgement" AND "require
training." Are these separate outcomes evaluated independently? Can they
fail independently? (e.g., acknowledgement tasks created successfully but
training tasks fail — what happens?)

**4. Are outcomes immediate or eventual?**

When a form is submitted, do outcomes fire synchronously (blocking the form
submission response) or asynchronously (queued for later processing)?
Synchronous is simpler but means a slow outcome blocks the user.
Asynchronous is more resilient but harder to report errors.

### Task Completion

**5. Who marks a task complete?**

When a form is submitted, how does the assignment service know the task is
done? Options:

- **The Form Service calls back** — form submission triggers `completeTask`.
  Form Service needs to know about the assignment service.
- **The UI calls both** — after form submission, the UI calls `completeTask`.
  Simple but relies on the client not failing between the two calls.
- **Event-based** — Form Service emits "form.submitted", a listener marks
  the task complete. Decoupled but needs event infrastructure.

The answer might differ per task type. Form tasks and acknowledgement tasks
might use one pattern; training tasks might use another.

**6. How does training completion flow back?**

Training lives in a separate platform (LMS) that we may not control. How
does the assignment service learn that Bob finished "Hazardous Materials
Training"?

- **The LMS calls back** — LMS has a webhook/API that calls `completeTask`
  when a course is finished. LMS needs to know about assignment.
- **Polling** — something periodically checks the LMS and updates tasks.
  Decoupled but introduces latency.
- **The UI bridges it** — after completing training, the user returns to the
  assignment UI and clicks "mark complete." Simple but relies on the user.
- **Event-based** — LMS emits "course.completed", a listener marks the task.

This is the same question as #5 but across a service boundary we don't
control. The answer might need to be different per consumer.

**7. What if a task is completed but the consumer doesn't confirm?**

If the UI calls `completeTask` but the form submission fails (or vice versa),
the task status and the actual work are out of sync. How is this reconciled?

- Retry logic?
- A periodic reconciliation job that checks task status against consumer
  status?
- Accept eventual consistency and let users manually fix mismatches?

### Task Model

**8. How does the reference schema work?**

Each action type has different reference data:

```
complete-form:     { templateGroupId }
acknowledge-form:  { formId }
complete-training: { courseId }
```

Is the reference a strongly-typed discriminated union (each action type has
a known shape)? Or is it a loosely-typed JSON blob that the assignment service
stores without validation? Tradeoffs:

- **Strongly typed** — safer, but requires assignment service changes for
  every new action type
- **Loosely typed (JSON)** — flexible, but no validation at the assignment
  layer. Consumers must handle bad data.

**9. Template version pinning**

When a CompleteFormTask is created, should it pin to a specific template
version? If the template is updated between assignment and completion, the
user might get a different form than intended.

- Store `templateId` (specific version) alongside `templateGroupId`?
- Let the consumer always use the latest? (Simpler, but risky)
- Let the caller decide? (Pass both, consumer chooses)

**10. Should a task carry a human-readable description of the action?**

The action + reference is enough for a consumer to act, but is it enough for
a UI to display? "complete-training / courseId: hazmat-101" isn't
user-friendly. Should the task also carry:

- A display name? (e.g., "Hazardous Materials Training")
- A description? (e.g., "Required due to chemical handling in Lab 3")
- A URL or deep link to where the user should go?

Or is this the consumer's responsibility to resolve from the reference?

### Fan-Out and Scope

**11. Who coordinates the fan-out?**

When an admin assigns assessments to all child folders, something needs to
iterate over folders and users. Options:

- **Folder Service** — knows the hierarchy, calls assignment per person
- **Frontend / BFF** — queries folders, then calls assignment
- **Assignment Service** — accepts "assign to folder children" and queries
  folders itself

The assignment service staying thin (Option 1 or 2) keeps it simple but
means the fan-out logic lives elsewhere.

**12. Bulk operations**

The fan-out use case creates many tasks at once. Should the assignment
service have a bulk `createTasks` endpoint? Or is creating them one at a
time sufficient?

- 3 labs with 5 people each → 15 tasks, one-at-a-time is fine
- 100 folders with 50 people each → 5,000 tasks, needs bulk or async

At what scale does this matter? Should it be synchronous or fire-and-forget?

**13. What about late-joiners?**

If a new member joins Lab 1 after the acknowledgement tasks were created,
do they automatically get an acknowledgement task? Or does someone need to
manually assign them?

This is a gap in any point-in-time fan-out approach. Options:
- **Manual** — someone notices and creates a task for the new member
- **Folder Service watches membership changes** — when a member is added,
  check if there are outstanding assignments for that folder and create tasks
- **Periodic reconciliation** — a job compares folder membership against
  existing tasks and fills gaps

**14. What about re-assignment?**

If a task is UNASSIGNED, can it be reassigned to someone else? Or is it a
new task? The current model doesn't support changing `assignedTo` — unassign
creates a new state, not a transfer.

### Tracking and Rollup

**15. How does "full completion" work as a derived status?**

The Program Manager wants to know: "is Lab 1 fully acknowledged?" This is
a derived state — all acknowledgement tasks for that folder are COMPLETED.
Should this be:

- **Computed on query** — count completed vs total tasks per folder
- **A status on a parent entity** — something tracks the rollup
- **A separate entity** — a "campaign status" record

Computing on query is simplest and doesn't require any new concepts. It
works as long as the number of tasks per folder is manageable.

**16. How do you track progress across different outcome types?**

Lab 3 has both acknowledgement AND training tasks. The Program Manager wants
to see Lab 3's overall compliance status. Is this:

- Separate queries per action type, displayed side by side?
- A combined view that shows all outstanding tasks regardless of type?
- A "compliance score" that aggregates across types?

**17. Should there be a "campaign" or "rollout" entity?**

When the admin assigns an assessment to all labs, is there a concept of a
"campaign" that groups all resulting tasks (assessments, acknowledgements,
training)? Useful for:

- "What's the overall completion rate for this rollout?"
- "Remind everyone with outstanding tasks from this rollout"
- "Cancel all tasks from this rollout"

This could be a `campaignId` on each task, or it could be implicit (query
by template + folder + time range).

### Notifications

**18. What goes in the notification link?**

The task carries enough data for a basic notification. But the link varies
by action type:

- Acknowledge-form → can link directly to the form (formId exists)
- Complete-form → no form yet. Link to the task? A landing page?
- Complete-training → link to the LMS course page?

Does the task store a URL, or does the notification service resolve it from
the action + reference?

**19. Should notifications be customizable per action type?**

"You have a new form to complete" vs "You have a training course to complete"
are different messages. Does the notification template come from:

- The action type? (Each action type has a default notification template)
- The caller? (Whoever creates the task provides the notification config)
- The task itself? (Task carries a `notificationTemplate` field)

**20. Should there be reminder notifications?**

If a task stays in ASSIGNED status for a long time, should the system send
reminders? If so:

- Who configures the reminder schedule? The admin? The template?
- Is this the assignment service's job or the notification service's job?
- Does the task carry reminder configuration, or is it a system-wide policy?
