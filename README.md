Summary: Implement "Apply Rates" shortcut for standard condition offers to bypass validation and confirmation steps

1. Business Context & Objective

Objective: Expedite the offer creation lifecycle for standard condition products by eliminating redundant revenue checks, manual multi-level validations, and client confirmation steps.

Process Impact: Modifies the primary offer generation BPD. Introduces a new Exclusive Gateway/Boundary Event routing the process token directly from the initial Drag & Drop User Task to the Apply Tariffs User Task.

References: N/A

2. Requirement Definition

As an Offer Preparer (Existing Authorized Participant Group)
I want an "Apply Rates" boundary event button on the drag-and-drop Coach that only appears when all configured product cards are set to "standard conditions"
So that I can route the process directly to the final tariff application step without viewing revenues or executing manual validation tasks.

3. Acceptance Criteria (BDD Format)

Scenario 1: System evaluates state and displays bypass button

Given: The process instance is at the initial drag-and-drop User Task.

When: The user configures all product cards within the drag-and-drop component to have the "standard conditions" switch selected.

Then: The server-side validation executes successfully and the "Apply Rates" button becomes visible on the Coach.

Scenario 2: System evaluates state and hides bypass button

Given: The process instance is at the initial drag-and-drop User Task.

When: One or more product cards within the drag-and-drop component have the condition switch set to "special conditions".

Then: The server-side validation executes and the "Apply Rates" button is completely hidden from the Coach UI.

Scenario 3: Process token bypasses intermediate tasks

Given: The "Apply Rates" button is visible and active.

When: The user clicks the "Apply Rates" button.

Then: The token exits the current User Task and routes directly to the "Apply Tariffs" User Task, completely bypassing the "Revenues" screen, "Validators" routing logic, and the "Client Confirmation" User Task.

Scenario 4: Audit trail records bypass execution

Given: The token arrives at the "Apply Tariffs" step via the standard conditions bypass.

When: The system writes the standard audit log for the tariff application.

Then: The audit log explicitly records the path taken, verifying that the standard condition criteria were met and the intermediate validation steps were legitimately bypassed.

4. System & Data Specifications

Affected Components: Offer Generation BPD, Drag-and-Drop Coach View (UI), Server-side Validation AJAX Service/Service Flow.

Data Mapping / Contracts: The isStandardCondition boolean (or equivalent flag) on the bound product Business Objects must be strictly evaluated server-side before rendering the button. The process context must retain these flags through the token jump.

State Transitions: Process instance lifecycle state transitions directly from Offer Preparation to Tariff Application, skipping Revenue Review, Validation Routing, and Awaiting Client Confirmation.

5. Out of Scope

Modifications to the existing concurrency/rework blocking logic (handled by existing architecture).

Changes to the routing rules or Coach UI of the Revenues, Validators, or Client Confirmation steps for non-standard offers.

6. Assumptions & Constraints

To be clarified during Sprint Planning: It is assumed that no specific, elevated authorization is required for this bypass; the Participant Group currently allowed to prepare new offers retains the right to execute this shortcut.
