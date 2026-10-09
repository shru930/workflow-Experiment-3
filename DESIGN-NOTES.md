# Design Notes

Supporting rationale for the three models in this repository. The
[README](README.md) explains *what* each process does and lists every failure
path. This document explains *why* the failure handling is built the way it is,
including alternatives that were considered and rejected.

---

## 1. How the failure paths were found

Each model was built in two passes.

1. **Happy path first.** One straight line from start to successful end, with
   every task typed.
2. **Interrogate every step.** For each task the same five questions were asked:
   - *What if the data is wrong?* (validation, KYC, address, documents)
   - *What if nobody answers?* (student, guide, committee, applicant, shipper, recipient)
   - *What if the answer is "no" or "not yet"?* (reject, revise, decline, counter-offer)
   - *What if the system is down?* (mail, scanner, credit bureau, payment gateway)
   - *What if someone pulls out half-way?* (team, guide, shipper, applicant)

Every "yes" became a row in the Failure Path Register and was then modelled. A
failure written in the register but missing from the diagram does not count.

---

## 2. Choosing the right construct for each failure

| Situation | Construct | Why not something else |
|---|---|---|
| A task's own result is bad (invalid form, score too low) | **Exclusive gateway** after the task | The task completed normally; the *data* decides the route. An error event here would hide a business outcome as if it were a technical fault |
| The task cannot complete (pickup impossible, scanner offline, bureau down) | **Error boundary event** | The worker throws a named error code (`PICKUP_FAILED`, `SCAN_FAILED`, `BUREAU_UNAVAILABLE`). The task is abandoned and the fallback starts, without a gateway after every task |
| A deadline passes and the work is no longer wanted | **Interrupting timer boundary** | Offer validity, submission window, revision window: when the timer fires, the original task must stop |
| A deadline passes but the work should continue | **Non-interrupting timer boundary** | Committee SLA: chasing the reviewers must not cancel their review. Assignment 1 uses two of these on one task (reminder at 5 days, HoD at 10) |
| Waiting for one of several outside events | **Event-based gateway** | Guide accepts / declines / silence; arrival / delay report / ETA; collected / hold expired. An exclusive gateway decides on data that already exists and cannot *wait* |
| Something outside interrupts a whole stage | **Message boundary on a sub-process** | Withdrawal during guide allocation, cancellation during transit: one boundary covers every task inside the stage |
| Something can interrupt *at any stage* | **Event sub-process** | Loan withdrawal can arrive anywhere from documents to disbursement. Putting a boundary on every task would need over 20 boundaries |
| Something already done must be undone | **Compensation** | Refund the shipping charge, refund the processing fee, reverse the loan account |
| A fatal finding must stop everything | **Terminate end event** | Confirmed fraud: the withdrawal event sub-process and any other token must die with the application |
| A retry | **Loop with a counter and a guard gateway** | Unbounded loops are the most common defect in exception models. Every loop here is bounded |

---

## 3. The inclusive gateways

`bpmnlint`'s recommended rule set warns against inclusive gateways, because an
inclusive *join* has to work out which branches are still able to arrive, and
that is easy to get wrong. They are used here because both cases are true
"any combination" decisions:

- **Shipment delay response.** The customer is told about every delay. Reroute
  only if the road is blocked, and a replacement vehicle only if the truck broke
  down. Both, one or neither may apply.
- **Loan extra checks.** Collateral valuation (secured loans), guarantor
  verification (when there is a guarantor) and enhanced due diligence (large
  amounts) are independent conditions.

Modelling either of these with exclusive gateways would need 2ⁿ branches.
Modelling them with a parallel gateway would run checks that do not apply. In
both models the split and the join are a matched pair inside the same scope,
with no other paths entering, so the join semantics stay simple. The loan split
has a default flow (*None*). The delay split has an always-true branch
(*Always → notify shipper*), so at least one token always flows.

---

## 4. Avoiding deadlock in parallel branches

Parallel blocks and failure paths do not mix well. If one branch of an AND-split
leaves through an end event, the AND-join waits forever for it.

**Assignment 3** runs four verification checks in parallel. None of them may end
the process from inside its branch:

- KYC mismatch goes to *Verify identity manually* and then **back into the
  branch** through a merge.
- Bureau failure goes to *Perform manual credit assessment* and back into the
  branch.
- Fraud screening only *records* a flag.

All four always reach the join. The decision (*Verification outcome?*) is made
once, after the join, when all the evidence is available.

**Assignment 2** first had the tracking notification running in parallel with
the whole transit stage. That was rejected. Transit can leave through four
boundary events (damage, loss, cancellation, scan error), and each of those would
have stranded a token at the join. The parallel block was moved to the short,
failure-free pair *Send tracking update* / *Plan route and book line-haul*.

**Assignment 1** merges the manual-notification fallback into the notification
branch with an exclusive merge *before* the AND-join. Wiring both into the
AND-join would make the join wait for two tokens on a branch that only ever
produces one.

---

## 5. Compensation versus "just another task"

Compensation is used only where an activity has **already completed** and its
effect must be reversed:

| Model | Completed activity | Compensation handler | Thrown by |
|---|---|---|---|
| 2 | Confirm booking and take payment | Refund charges minus fee | Pickup finally fails; cancellation after dispatch |
| 3 | Collect processing fee | Refund processing fee | Withdrawal event sub-process |
| 3 | Create loan account | Reverse loan account | Withdrawal; disbursement finally fails |

Assignment 1 first used compensation too: *Release provisional hold* was the
handler for *Place provisional hold on guide*, thrown inside the allocation
sub-process. Running the model on Camunda 8.8 showed that a compensation throw
**inside an embedded sub-process** never starts its handler; the instance stops
at the throw event. A minimal two-task test confirmed it: the same pattern at
process level completes, but inside a sub-process it hangs. The release is
therefore an ordinary service task on the decline / timeout path. All the
remaining compensation in Assignments 2 and 3 is thrown at process level (or
from the event sub-process), where Camunda executes it correctly.

Where nothing has completed yet, an ordinary task is clearer. For example, the
withdrawal during guide allocation uses *Cancel request and free guide slot*
because the sub-process was interrupted, not completed. Compensation of an
interrupted scope is not defined by BPMN, so relying on it there would be wrong.

The handlers are linked to their activities by **associations**, not sequence
flows, and carry `isForCompensation="true"`. They run only when compensation is
thrown, and only for activity instances that actually completed. A withdrawal
before the fee was taken therefore refunds nothing, which is correct.

---

## 6. Retries: engine versus model

Two kinds of retry appear, and they are deliberately different.

- **Technical retries** (mail server, credit bureau, payment gateway, database)
  use Zeebe's job retries (`retries="3"` on the task definition). The worker
  retries quietly. Only when the retries run out does it throw a BPMN error,
  which the model catches with an error boundary. Drawing three explicit retry
  loops would clutter the diagram with infrastructure detail.
- **Business retries** (resubmit, revise, reschedule pickup, re-attempt
  delivery, re-send agreement, approach the next guide) are visible loops,
  because a person or an outside party is involved, time passes, and the
  business wants to count them. Each is bounded by a counter.

---

## 7. Pools versus lanes

The brief lists participants as "lanes / pools", and the choice was made per
participant:

- **Lanes** for everyone inside the organisation who is directed by the process
  (coordinator, committee, hub staff, underwriters, operations, the system
  itself).
- **Pools** for independent parties whose behaviour the process cannot
  control and can only exchange messages with: shipper, recipient, applicant,
  credit bureau, and the faculty guide (who may freely accept or decline).

This is also a technical requirement. Message flows are only legal *between*
pools, and the event-based gateways that wait for "accept / decline / silence"
need real messages to wait for.

---

## 8. Alternatives considered and rejected

- **One big end event per model.** Rejected for the same reason as in Exercise 1.
  *Rejected*, *lapsed*, *withdrawn*, *lost* and *returned* are different business
  outcomes, and the organisation will want to count each one.
- **Boundary events on every task for withdrawal / cancellation.** Rejected in
  favour of one boundary on a sub-process (Assignments 1 and 2) or one event
  sub-process (Assignment 3).
- **Error end events at the top level** (e.g. *Invalid team* as an error end).
  Rejected. In Camunda 8 an error that nothing catches raises an incident
  instead of ending the instance cleanly. Error ends are used only inside
  sub-processes, where a boundary catches them. Top-level failure outcomes are
  plain end events with descriptive names.
- **Collapsed sub-processes.** Rejected for this submission. The failure
  handling *inside* the guide allocation, hub transit and compliance stages is
  part of what is being assessed, so it is kept visible on the main diagram.
