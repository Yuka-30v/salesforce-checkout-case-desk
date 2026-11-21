# Checkout Case Desk (Salesforce)

A Salesforce Service app for logging, routing and escalating checkout issues in a retail store, built on the standard **Case** object.

> Status: in progress (started November 2026)

## 1. The problem

At the supermarket where I work, customers' loyalty app or loyalty card often fails to scan at the self-checkout machines. I first noticed it after a machine software update.

Today the workaround is:
- staff enter the card number by hand (I estimate 20+ times per shift),
- the fault is reported verbally to a supervisor,
- nothing is recorded, so nobody can see how often it happens, at which machines, or whether it is getting better or worse.

**Goal:** give staff a fast way to log each issue, route it to the right people, escalate urgent ones automatically, and give supervisors data instead of anecdotes.

## 2. Users

| Role | What they need |
|---|---|
| **Checkout Agent** | Log an issue in under a minute; see and update their own cases |
| **Supervisor** | See all the team's cases; be alerted when an urgent case is left untouched; see trends by store, type and machine |

## 3. Requirements

| ID | Requirement | Priority |
|---|---|---|
| R1 | An agent can log a checkout issue as a Case with Touchpoint, Store, Machine, Payment Method, Error Code and a description | Must |
| R2 | Cases are split into three types: **Payment**, **Loyalty**, **Device** | Must |
| R3 | New cases are routed automatically to the right queue by type | Must |
| R4 | Error Code is required when the case reason is a payment failure | Must |
| R5 | A High-priority case left untouched for 4 hours creates a task for the Supervisor | Should |
| R6 | Customers can report a problem by email, which creates a case and sends an automatic reply | Could |
| R7 | A dashboard shows open cases by type, average case age, cases by store and cases by machine | Must |
| R8 | Agents see only their own cases; Supervisors see the whole team's | Must |

## 4. Planned data model

Standard object: **Case**. Custom fields:

| Field label | API name | Type | Values / notes |
|---|---|---|---|
| Touchpoint | `Touchpoint__c` | Picklist | Self-checkout, Staffed till, Loyalty app |
| Store | `Store__c` | Picklist | Store A, Store B, Store C, Store D |
| Machine | `Machine__c` | Lookup (Machine) | Links each case to one machine record, so counts per machine are reliable. Machines that have cases can't be deleted, which protects the history. |
| Payment Method | `Payment_Method__c` | Picklist | Card, Contactless, Cash, Loyalty points |
| Error Code | `Error_Code__c` | Text | Required for payment failures (R4) |

Record types on Case: **Payment**, **Loyalty**, **Device**.

Standard field changes:
- add a **Payment Failure** value to **Case Reason** (used by R4)
- add an **In store** value to **Case Origin**, because most cases are logged by staff in the store, not by phone, email or web

Custom object: **Machine** (`Machine__c`), one record per checkout machine.

| Field label | API name | Type | Values / notes |
|---|---|---|---|
| Machine Name | `Name` | Text | e.g. SCO-03 |
| Store | `Store__c` | Picklist | Store A, Store B, Store C, Store D |

Store values come from a Global Value Set, shared by Case and Machine.
A lookup filter on Case.Machine__c only shows machines from the case's store.


## 5. Design decisions

### Why the standard Case object
In my Dynamics 365 version (Checkout Issue Tracker), I built two custom tables,
Checkout Issue and Machine, and created my own Issue status column.

For Salesforce I used the standard Case object instead. It already comes with
status, priority, owner, queues, assignment rules, escalation rules and
Email-to-Case. That means most of my requirements (R3, R5, R6) are met with
configuration rather than custom build, which keeps the org simpler to maintain
and closer to how a real support team uses Service Cloud.

### Case Origin vs a custom "Touchpoint" field
Case already has a standard field, Case Origin, so before adding anything I
checked whether it could do the job. It can't: Origin records *how* a problem
was reported (phone, email, web), while I also need *where* the fault happened
(self-checkout, staffed till, loyalty app). 
A customer can email about a
self-checkout fault, so the two values are independent.

Decision: keep Case Origin for the reporting route, and add one custom picklist,
Touchpoint (`Touchpoint__c`), for where the fault happened. I renamed my
original "Channel" field to Touchpoint because "channel" is easily confused
with Origin.

### Machine: why a custom object instead of the standard Asset
Case already links to the standard **Asset** object, so I checked it first.
Asset is designed to track products that customers have bought and installed,
and every asset must belong to an Account or Contact. In this project the
machines belong to the store, not to a customer. Using Asset would mean
registering stores as Accounts, which would mix real customers with stores
and skew reports such as the total number of customers.

If I were building this for a self-checkout manufacturer supporting
supermarket chains as its customers, Asset would be the right choice.

Decision: use a custom **Machine** object, as in my Dynamics version.

### Keeping Store consistent between Case and Machine
In my Dynamics version, staff typed machine names as free text, so the same
machine was recorded as both "SCO-1" and "SCO 1" and the counts per machine
were wrong. I fixed it by creating a Machine table and linking issues to it
with a lookup. The Machine lookup here prevents that problem from the start.

Storing Store on both Case and Machine creates two new risks:
1. The two store lists drift apart (for example, a new store added to Case
   but not to Machine).
2. A single case contradicts itself: the case says Store A, but the machine
   selected belongs to Store B.

I keep Store on Case because some cases, such as loyalty app issues, have
no machine. I considered a formula field that copies Store from the Machine,
but rejected it because Store would be empty whenever no machine is linked.

Decision: a **Global Value Set** defines the store list once for both
objects (risk 1), and a **lookup filter** on Case only shows machines from
the case's store (risk 2).

## 6. Out of scope

- Fixing the machines themselves (cases record and route the fault; repairs happen elsewhere)
- Integration with the real checkout system (sample data only)

## 7. Build progress

- [x] Store global value set and Machine object
- [x] Custom Case fields, Machine lookup (with lookup filter) and record types (R1, R2)
- [ ] Page layouts, Lightning app, list views
- [ ] Security model: Agent and Supervisor users, roles, Case OWD = Private (R8)
- [ ] Queues and assignment rules (R3)
- [ ] Validation rules (R4)
- [ ] Escalation flow (R5)
- [ ] Email-to-Case (R6)
- [ ] Reports and dashboard (R7)
- [ ] Sample data import and UAT
- [ ] Dynamics vs Salesforce comparison

## Related

- Dynamics 365 version: [dynamics-checkout-issue-tracker](https://github.com/Yuka-30v/dynamics-checkout-issue-tracker)
