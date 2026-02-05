# LHAT Assignment Workflows

## 1. Ergo Evaluation: Ergo Admin assigns Ergonomist to complete Ergo Eval on Subject

```mermaid
flowchart LR
    A(("Lab Safety Admin
    notified PI Needs to
    complete LHAT"))
    B[["Lab Safety Admin
    creates Lab group
    (folder) for Lab"]]
    C{"Viewing Folder
    or Viewing
    Program Overview"}
    D["Lab Safety Admin Views
    Program Overview Page"]
    E["Lab Safety Admin
    Searches for PI by name
    or email"]
    F["Lab Safety Admin Selects
    'LHAT' Template to
    assign to PI"]
    G(("Lab Safety Admin
    assigns LHAT to PI"))
    H("Lab Safety Admin
    selects Assign")

    A --> B --> C
    C -- Program Overview --> D --> E --> F --> G
    C -- Folder --> H --> F
```

## 2. Ergo Self-Assessment: Ergonomist Assigns Self-Assessment to Employee

```mermaid
flowchart LR
    A(("Lab Safety Admin
    notified PI Needs to
    complete LHAT"))
    B[["Lab Safety Admin
    creates Lab group
    (folder) for Lab"]]
    C{"Viewing Folder
    or Viewing
    Program Overview"}
    D["Lab Safety Admin Views
    Program Overview Page"]
    E["Lab Safety Admin
    Searches for PI by name
    or email"]
    F["Lab Safety Admin Selects
    'LHAT' Template to
    assign to PI"]
    G(("Lab Safety Admin
    assigns LHAT to PI"))
    H("Lab Safety Admin
    selects Assign")

    A --> B --> C
    C -- Program Overview --> D --> E --> F --> G
    C -- Folder --> H --> F
```

## 3. LHAT: Lab Safety Admin needs to assign LHAT to PI

```mermaid
flowchart LR
    A(("Lab Safety Admin
    notified PI Needs to
    complete LHAT"))
    B[["Lab Safety Admin
    creates Lab group
    (folder) for Lab"]]
    C{"Viewing Folder
    or Viewing
    Program Overview"}
    D{"Search PI
    or Lab Group?"}
    E["Lab Safety Admin Views
    Program Overview Page"]
    F["Lab Safety Admin
    Searches for PI by name
    or email"]
    G["Lab Safety Admin Selects
    'LHAT' Template to
    assign to PI"]
    H["Lab Safety Admin
    assigns LHAT to PI"]
    I(("LHAT is assigned to PI"))
    J("Lab Safety Admin
    selects Assign")
    K["Email sent to PI to create
    LHAT for their folder"]

    A --> B --> C
    C -- Program Overview --> D --> E --> F --> G --> H --> I
    C -- Folder --> J --> D
    H --> K
```

## 4. LHAT: Lab Safety Admin needs to assign LHAT to Lab Group

```mermaid
flowchart LR
    A(("Lab Safety Admin
    notified Lab Needs to
    complete LHAT"))
    B[["Lab Safety Admin
    creates Lab group
    (folder) for Lab"]]
    C{"Viewing Folder
    or Viewing
    Program Overview"}
    D["Lab Safety Admin Views
    Program Overview Page"]
    E["Lab Safety Admin
    Searches for Lab"]
    F["Lab Safety Admin Selects
    'LHAT' Template to
    assign to PI"]
    G(("Lab Safety Admin
    assigns LHAT to PI"))
    H("Lab Safety Admin
    selects Assign")
    K["Email sent to PI to create
    LHAT for their folder"]

    A --> B --> C
    C -- Program Overview --> D --> E --> F --> G
    C -- Viewing Folder --> H --> F
    G --> K
```
