# Expense Claims Portal

An online replacement for a paper staff expense claim form. Staff sign in with an emailed one-time code, add any number of expense lines with receipts from a phone or PC, and submit. Approvers review the full claim from a secure link in their email, with no sign-in, and can approve, decline or ask a question.

> Case study. The production code, flows and data belong to my employer and are not published. This repository documents the design, architecture and decisions. All names and values shown are generic.

## Requirements

- Public page, no Entra sign-in, but only staff can use it. The staff check runs against the existing all-staff group, not a hand-kept whitelist.
- Multiple expenses of different types in one claim, each flagged billable or not, with GST.
- Travel and mileage with round trip, one way and multiple stops.
- Receipts from the phone camera, drag and drop, or scanned images and PDFs.
- Approval without a separate app: the approver opens the claim from email, can download it as a PDF at any time, and can ask the claimant for more information.
- An admin area that lists every claim and lets super admins edit settings.

## Architecture

```mermaid
flowchart LR
    subgraph Portal["Power Pages site"]
        UI["Claim form<br/>My claims<br/>Admin area"]
    end
    subgraph Flows["Power Automate"]
        F1["Send OTP / Verify OTP"]
        F3["Claim, line, receipt<br/>and note actions"]
        F8["Approval routing<br/>and notifications"]
        F9["Admin"]
    end
    subgraph SP["SharePoint"]
        L1[("Claims")]
        L2[("Lines")]
        L3[("OTP")]
        L4[("Config")]
        L5[("Notes")]
        D1[("Receipts library")]
    end
    G["Entra ID<br/>all-staff group"]
    UI --> F1 & F3 & F9
    F1 --> G
    F1 --> L3
    F3 --> L1 & L2 & D1 & L5
    F3 --> F8
    F9 --> L4
```

Nine flows in total, eight of them HTTP-triggered from the page.

## Claim lifecycle

```mermaid
stateDiagram-v2
    [*] --> Draft
    Draft --> Submitted: claimant submits
    Submitted --> Approved: approver approves
    Submitted --> Declined: approver declines
    Submitted --> MoreInfo: approver asks a question
    MoreInfo --> Submitted: claimant replies or edits and resubmits
    Approved --> [*]
    Declined --> [*]
```

## Sign-in and staff check

```mermaid
flowchart TD
    A["Enter work email"] --> B{"Member of<br/>all-staff group?"}
    B -->|no| C["No access message"]
    B -->|yes| D["Email one-time code"]
    D --> E["Enter code"]
    E --> F{"Valid and<br/>not expired?"}
    F -->|no| E
    F -->|yes| G["Session starts"]
```

## Approval by secure link

When a claim is submitted the flow stores a review token on the claim and emails the approver a link that carries it. The link opens that one claim in the portal with no sign-in, so approvers never need an account or an app.

```mermaid
sequenceDiagram
    participant C as Claimant
    participant P as Portal
    participant F as Flows
    participant A as Approver
    C->>P: Submit claim
    P->>F: Submit
    F->>F: Generate review token
    F-->>A: Email with tokened link
    A->>P: Open link
    P->>F: Load claim (token checked)
    A->>P: Approve / Decline / Ask a question
    P->>F: Review action (token checked)
    F-->>C: Outcome email
```

## Decisions worth noting

- No OCR. Entry is manual with the receipt attached, which kept the build simple and accurate.
- No paid mapping keys. Staff open the route in a maps app and enter the kilometres; the provider hooks stay in the code but are switched off.
- PDF receipts are converted to images in the browser with PDF.js, so receipt pages appear inside the claim document without a paid PDF merge service.
- Lists were created by CSV import, so the flows reference the generated internal column names. Each one was checked against the import before wiring.
- Page fonts use the system font stack so the site looks native on any device.

## Documentation delivered

A user guide for staff and accounts with screenshots, and a technical note for whoever supports it next.

## Stack

Power Pages, Power Automate (nine flows), SharePoint Online lists and library, Entra ID groups, Outlook, PDF.js.
