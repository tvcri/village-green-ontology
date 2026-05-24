# Village Green — Ontology Summary
*Working document — session summary as of May 2026, v2*

---

## Context and Boundaries

Village Green (VG) is an open-source platform intended to replace Club Express (CE) for managing Village operations at The Village Common of Rhode Island (TVCRI) and its local spokes. It is **not a CRM** — it is an operational system that intersects with the CRM (currently Little Green Light / LGL). VG will push and pull data to/from LGL but is not a replacement for it.

**Current scope:** Village daily operations — specifically the onboarding of Members and Volunteers, and the pairing of member service requests to volunteers. Natural persons only; no organizational persons in this domain.

**Out of current scope (but not precluded):** Events management, fundraising, donor tracking. These belong to LGL or a future dedicated system. VG's Person model is designed to be extensible to events (via stable pId as a future FK) without modeling events now.

**CRM alignment:** LGL's central entity is the Constituent, which maps to VG's Person. LGL exposes `external_constituent_id` — the intended hook for cross-system sync. LGL's event model is fundraising-oriented and unsuitable for Village community events.

**Transition period:** CE will be sunset incrementally. During transition, VG will need to push data into CE. CE has no API; a Playwright-based form automation bridge is the likely approach, treated as a temporary artifact not shaping VG's data model.

---

## Ontological Entities

### Person
The stable center of gravity. Every person in the system is a Person. Roles hang off Person; Person does not duplicate role-specific data.

**Properties (facts about the person):**
- Names: first, middle initial, last, preferred/nickname
- Preferred pronouns
- Date of birth
- Gender
- Veteran status
- Preferred language (single value) ← the language this person communicates in
- Languages spoken (set, other than English) ← operationally relevant for service matching and communications
- Street address
- Contact methods: landline, cell, email (typed, multiple)
- Accessibility needs: hearing, vision, mobility (walker/cane/wheelchair)
- Emergency contacts (structured sub-entities): name, relationship, phone(s), email (optional)
- Village association (reference to Village) ← current Village only; no history tracked; nullable for hub-level staff

**Notes:**
- A Person who is only a User may have null address and other fields — acceptable, meaningful nulls
- Aligns with LGL Constituent; `lgl_constituent_id` should be carried for sync purposes
- Language properties belong at Person level because they are facts about the person regardless of role, and are relevant to both service matching (volunteer side) and communication preferences (member side)
- Emergency contacts are structured (not free text) because operational use — however rare — requires actionable data
- Village association is current state only — history of village membership is not tracked in the current model. This is a known simplification.

---

### User *(role)*
A Person who can access the VG system.

**Properties:**
- Username
- Credential/auth info
- Access level (includes "staff" as a level, not a separate role)

---

### Member *(role)*
A Person who can request services from the Village.

**Properties:**
- Status history (pending → active → lapsed → active...) — VG is authority for this history
- Circle of Pride: member flag, service preference flag
- Newsletter delivery preference (email vs. printed mail)
- Invoice delivery preference (email vs. printed mail)
- Dues arrangement (via Household if dual; directly if individual)
- Service eligibility status — distinct from membership standing; a member may be ineligible for certain services if needs exceed Village capabilities. Policy in flux as board considers open membership model.

---

### Volunteer *(role)*
A Person who can provide services to Members.

**Properties:**
- Status history (pending → active → inactive → active...) — VG is authority for this history
- Service capabilities (set, drives the service-pairing logic):
  - Driver – Rides
  - Driver – Errands
  - Light Household Maintenance
  - Steering Committee
  - Technology Support
  - Village Friends – Check-In Calls
  - Village Friends – Friendship Calls
  - Village Friends – Friendship Visits
- Circle of Pride: volunteer support flag
- Employment status, occupation (captured for Village Friends volunteers)
- Friendship visit activities/interests (free text, optional)
- Driving credentials validated flag (set at graduation from PendingVolunteer; documents archived via onboarding requirements)

---

### Household *(thin grouping entity)*
Links two Members sharing a dwelling and a dues obligation. Dual Household is a billing convenience, not an ontological unit — two Persons, two Members, one dues arrangement.

**Properties:**
- Reference to Member 1, Member 2
- Dues arrangement (level, payment method, frequency)

**Scope:** Appears only in membership dues context. No role in service requests, communications, or events.

---

### Village *(first-class entity)*
A spoke of TVCRI. The organizational unit within which Members receive services and Volunteers provide them. A Person belongs to exactly one Village at a time (current association only — no history tracked).

**Properties:**
- Name
- Status (active / inactive)
- Service area (free text — geographic description of the Village's coverage)
- Contact methods: phone, email (village-level, not inherited from any individual)

**Notes:**
- Village has no coordinator property — the coordinator is modeled as a VillageRole, keeping Village clean and avoiding denormalization
- Person carries the village association as a nullable reference — null is valid for hub-level staff not associated with a specific Village
- The 1:n relationship (one Village, many Persons) is implicit in the village reference on Person

---

### VillageRole *(junction entity — Person × Village)*
A named role that a Person holds within a specific Village. Drives operational responsibilities and will inform role-based access control (RBAC) in a future session. A Person may hold more than one VillageRole within a Village, or roles across multiple Villages.

**Properties:**
- Person reference
- Village reference
- Role type (enumeration — to be defined in collaboration with Caroline and the working group; expected values include coordinator, ambassador, steering committee, volunteer lead, and others)
- Status (active / inactive)
- Effective date

**Notes:**
- Role type enumeration is intentionally left open — it is an operational question for the working group, not an ontology decision to make in abstraction
- VillageRole history is not tracked — current status only. This is a known simplification consistent with how Village membership is handled on Person
- RBAC implications (what each role type is permitted to do in VG) are deferred to a dedicated session

---

### PendingMember *(transient, pre-role)*
A Person who has expressed interest in membership and is undergoing assessment. Not yet a Member. Exists until graduation (→ Member) or rejection. Record persists permanently after either outcome.

**Properties:**
- Application date
- Membership Ambassador (reference to Volunteer or staff User)
- Household type: Single or Dual
- Ambassador notes (free text — retrospective record of the Ambassador visit, which is a pre-entry gate not tracked in-system)
- Waiver signed flag, signature date
- Dues level selected, payment method selected
- DuesRequirement (single owned sub-entity — the sole in-system graduation gate)
- Graduation event reference (links to resulting Member record)
- Rejection/withdrawal event reference

**Notes:**
- The Ambassador visit clears before the record is created; it is not a tracked workflow step within VG
- The only in-system graduation gate is dues clearance, recorded via DuesRequirement and DuesEvidence
- Policy in flux: board considering open membership without dues gate; service eligibility assessment may become a distinct step
- Most contact/personal info collected here graduates to Person on approval

---

### PendingVolunteer *(transient, pre-role)*
A Person who has submitted a volunteer application and is progressing through onboarding. Not yet a Volunteer. Exists until graduation (→ Volunteer) or withdrawal.

**Onboarding is not a fixed linear pipeline** — it is a dynamic checklist of OnboardingRequirements, shaped by the service capabilities the applicant has selected. Checklist completion drives progression.

**Properties:**
- Application date
- Volunteer Ambassador (reference)
- Service capabilities expressed interest in (determines which OnboardingRequirements are generated)
- Waiver signed flag (confidentiality + liability), signature date
- Background check: authorization date, completion date, outcome
- Orientation: date, format (Zoom/in-person)
- Badge: photo received, lanyard preference (standard blue / rainbow Circle of Pride)
- Vehicle records (if Driver capability selected): model, make, year, plate, state, insurance company, license restrictions
- Ambassador/staff notes

---

### OnboardingRequirement *(owned by PendingVolunteer)*
A single checklist item required for a specific volunteer capability. Generated dynamically based on capability selections.

**Properties:**
- Requirement type (e.g. background check authorization, driver's license copy, vehicle registration, insurance proof, orientation attendance)
- Triggering capability (e.g. Driver–Rides, Village Friends)
- Status: outstanding / satisfied / waived
- Satisfied date

**Relationship:**
- An OnboardingRequirement may be satisfied by one or more **RequirementEvidence** items

---

### OnboardingEvidence *(owned by OnboardingRequirement)*
The record of how a specific OnboardingRequirement was satisfied. Separates the *fact* of satisfaction from the *means* of satisfaction, allowing the evidence model to evolve independently.

**Properties:**
- Evidence type: *attestation* (staff confirms verbally or visually, e.g. checkbox) or *document* (a stored artifact)
- Recorded by (reference to staff User)
- Recorded date
- Document reference (present if evidence type is document; points to stored artifact — blob storage or external reference depending on implementation)
- Notes (optional, free text)

**Notes:**
- Currently, a staff member checking a box is an attestation-type evidence item. A scanned insurance card is a document-type evidence item. Same ontological relationship, different evidence types.
- This entity exists only in the PendingVolunteer context.
- The document storage implementation (blob, file path, external URL) is a schema-level decision deferred to implementation.

---

### DuesRequirement *(owned by PendingMember)*
The single graduation gate for PendingMember — records whether initial dues have been received. Parallel in structure to OnboardingRequirement but scoped to the membership context. Not a checklist; there is exactly one DuesRequirement per PendingMember.

**Properties:**
- Status: outstanding / satisfied / waived
- Satisfied date

**Relationship:**
- A DuesRequirement may be satisfied by one or more **DuesEvidence** items

---

### DuesEvidence *(owned by DuesRequirement)*
The record of how dues clearance was established. Currently staff attestation; designed to accommodate external transaction references (QuickBooks, PayPal) in future iterations. Parallel in structure to OnboardingEvidence.

**Properties:**
- Evidence type: *attestation* (staff confirms receipt, e.g. checkbox) or *transaction* (external payment reference)
- Recorded by (reference to staff User)
- Recorded date
- Transaction reference (present if evidence type is transaction; external payment ID or reference — implementation detail deferred)
- Notes (optional, free text)

**Notes:**
- The fact/means separation is the same pattern as OnboardingEvidence: DuesRequirement records that dues were cleared; DuesEvidence records how.
- Named distinctly from OnboardingEvidence because the ownership context and future extension path differ, even though the current structure is similar.

---

## Key Design Decisions

1. **Roles pattern:** Person is the person; roles (User, Member, Volunteer) are separate entities keyed by pId. Role membership is implicit in the existence of a role record, not stored as an array on Person.

2. **Address on Person:** Address is a fact about the person, not the role. Nulls for User-only persons are acceptable and meaningful.

3. **Accessibility needs on Person:** Not Member-only — operationally relevant for volunteers too. Designed for current state but correct at the person level.

4. **Language on Person:** Both `preferredLanguage` (single) and `languagesSpoken` (set) belong on Person, not on a role. Language is a fact about the person relevant to matching (volunteer capabilities) and communication (member preferences) regardless of how the record is surfaced.

5. **Emergency contacts on Person:** Structured sub-entities, not free text. Rare operational use still requires actionable data (a phone number, not a note). Belongs on Person because it does not vary by role.

6. **Status history owned by VG:** VG is the authority for Member and Volunteer role history (active/inactive periods with dates). LGL may be informed but does not own this.

7. **Pending as pre-role:** PendingMember and PendingVolunteer are distinct transient entities, not states on Member/Volunteer. A Person is not a Member or Volunteer until graduation.

8. **Driver as capability bundle, not sub-role:** Driving credentials are validated once at onboarding and archived via RequirementEvidence. No ongoing credential lifecycle management.

9. **Household is thin:** Exists solely to express shared dues between two Members. Does not extend into service, communications, or events.

10. **Dynamic onboarding checklist:** PendingVolunteer onboarding is modeled as a set of OnboardingRequirements derived from capability selections, not a fixed pipeline.

11. **Requirement vs. evidence separation (both contexts):** In the volunteer context, OnboardingRequirement records what is needed; OnboardingEvidence records how it was satisfied. In the member context, DuesRequirement records the dues gate; DuesEvidence records how clearance was established. The pattern is consistent: fact of clearance is separated from means of clearance, allowing each evidence model to evolve independently.

12. **Pending records persist permanently:** Graduation is an event, not a transformation. A new role record (Member or Volunteer) is created at graduation; the pending record is closed and preserved as a historical artifact. PendingMember and PendingVolunteer records are never purged — they are the audit trail for how a person entered the system, including rejection and withdrawal outcomes.

13. **Ambassador visit is pre-entry, not in-system:** The Membership Ambassador visit clears before a PendingMember record is created. VG does not track the visit as a workflow step; Ambassador notes on PendingMember are retrospective documentation only.

---

## Entities Not In Scope (Current Phase)

- Organizations / non-natural-person persons
- Events, registration, attendance
- Service requests and volunteer pairing *(next major domain)*
- Financial transactions, dues payment processing (LGL/external responsibility)
- Detailed communications history (LGL responsibility)

---

## Decision Log

A chronological record of significant ontological decisions and the reasoning behind them. Useful for explaining the model to new collaborators or reconstructing why alternatives were rejected.

- **May 2026 — Person as roles container:** Adopted roles pattern (User, Member, Volunteer as separate entities keyed by pId) rather than a flags-on-Person approach. Rationale: a person can hold multiple roles simultaneously or sequentially; role-specific data stays with the role, not the person.

- **May 2026 — Pending as pre-role, not state:** PendingMember and PendingVolunteer modeled as distinct transient entities rather than status values on Member/Volunteer. Rationale: a pending person is ontologically different from a member or volunteer — they have not yet been granted the role. Conflating them would blur the graduation boundary.

- **May 2026 — Household is thin:** Considered richer Household model (shared address, communications, events). Rejected for current scope. Household exists solely as a billing convenience linking two Members to a shared dues arrangement.

- **May 2026 — Dynamic onboarding checklist over fixed pipeline:** PendingVolunteer onboarding modeled as a generated set of OnboardingRequirements rather than a fixed stage pipeline. Rationale: requirement set varies by capability selection, and real-world onboarding involves back-and-forth reentry that a linear pipeline would misrepresent.

- **May 2026 — Language on Person, not role:** Both `preferredLanguage` and `languagesSpoken` placed on Person rather than Member or Volunteer. Rationale: language is a fact about the person, not a role attribute. Relevant to service matching (volunteer side) and communication preferences (member side); should not be duplicated or omitted depending on which role surface is used.

- **May 2026 — Emergency contacts structured on Person:** Modeled as structured sub-entities (name, relationship, phone(s), optional email) rather than free text on pending records. Rationale: however rarely used, operational need requires actionable data. Placed on Person because the information does not vary by role.

- **May 2026 — Requirement vs. evidence separation:** OnboardingRequirement records what is needed and whether satisfied. RequirementEvidence records how satisfaction was established (attestation vs. document). Rationale: current process uses staff attestation (checkbox); future may use document blobs. Separating the two allows the evidence model to evolve without touching the requirement model. This pattern is intentionally absent from PendingMember.

- **May 2026 — LGL event model rejected for Village events:** LGL's event model is fundraising-oriented (donation attribution with a date). Unsuitable for Village community events. Events domain deferred to a future dedicated system; VG's stable pId on Person is sufficient future-proofing for event registration FKs.

- **May 2026 — Requirement vs. evidence separation extended to PendingMember:** The fact/means separation established for volunteer onboarding (OnboardingRequirement / OnboardingEvidence) was applied to member onboarding as DuesRequirement / DuesEvidence. The entities are named distinctly because their ownership context and future extension paths differ (documents vs. external transaction references), even though the current structure is similar.

- **May 2026 — Pending records persist permanently:** Graduation is an event, not a transformation. A new role record is created at graduation; the pending record is closed and preserved. Rationale: pending records are the audit trail for how a person entered the system. Rejection and withdrawal records also persist for institutional memory.

- **May 2026 — Ambassador visit is pre-entry:** The Membership Ambassador visit clears before a PendingMember record is created. VG does not model or track the visit as a workflow step. Ambassador notes on PendingMember are retrospective documentation of a gate already cleared.

- **May 2026 — RequirementEvidence renamed OnboardingEvidence:** Renamed for specificity, to make the volunteer onboarding context explicit and distinguish it cleanly from DuesEvidence in the member context.

- **May 2026 — Village as first-class entity:** Village modeled as a first-class entity with operational properties, not a simple lookup value or label on Person. Persons belong to exactly one Village (current association, no history).

- **May 2026 — No coordinator property on Village:** Coordinator is a VillageRole, not a dedicated property on Village. Avoids denormalization and the question of what happens when the coordinator changes.

- **May 2026 — VillageRole history not tracked:** VillageRole records current state only. Consistent with how Village membership on Person is handled. A known simplification — adding ended_date later is a trivial schema migration.

- **May 2026 — RBAC deferred:** VillageRole will inform role-based access control but the permissions model is deferred to a dedicated session. The ontology establishes what roles exist; RBAC defines what they are permitted to do.

---

## Open Questions

- Will service eligibility assessment become a formal step in PendingMember onboarding as board policy evolves? *(design for extensibility; do not model prematurely)*

- **User as Person-backed only — service accounts not modeled.** The current model requires every User to be a Person, which forecloses non-person actors (automated sync jobs, API integration accounts, kiosk logins). For current scope this is appropriate — everyone who touches VG is a natural person, and temporary automation artifacts (LGL sync, CE bridge) can run under a designated staff User record without ontological pollution. If VG ever needs true service accounts, the clean fix is a thin **Principal** abstraction (a thing that can authenticate) with Person-backed User as one kind and ServiceAccount as another. Do not model this prematurely.

- **VillageRole type enumeration undefined.** The set of role types (coordinator, ambassador, steering committee, volunteer lead, others) is intentionally left open pending input from Caroline and the working group. This enumeration must be settled before schema work begins on VillageRole.

- **RBAC session needed.** VillageRole will inform role-based access control — what each role type is permitted to see and do in VG. This is a distinct session deferred until the ontology is more settled.

---

## Note for next session — MySQL schema translation

The ontology is sufficiently settled to begin a MySQL schema. The following translation decisions will need to be worked through:

**Identity and keys.** Every entity should have a surrogate `pId` (UUID or auto-increment — decide once and apply consistently). Person is the anchor; all role and pending records carry a `person_id` FK. Do not use names or emails as natural keys — the Hub data has shown these are not stable.

**Roles as separate tables.** User, Member, and Volunteer each become their own table with a `person_id` FK and their own columns. Role membership is implicit in row existence, not a flag on Person.

**Status history.** Member and Volunteer status history should be modeled as child tables (`member_status_history`, `volunteer_status_history`) with `status`, `effective_date`, and `ended_date` — not as a single status column on the role table. VG is the authority for this history.

**Pending records.** PendingMember and PendingVolunteer are permanent tables — records are never deleted, only closed via graduation or rejection event references. A `closed_at` timestamp and `outcome` enum (graduated / rejected / withdrawn) are the right closure pattern.

**Evidence pattern.** OnboardingRequirement → OnboardingEvidence and DuesRequirement → DuesEvidence follow a consistent parent/child pattern. Evidence type should be an enum (`attestation`, `document` for onboarding; `attestation`, `transaction` for dues). Document and transaction references are nullable columns — present only when the evidence type warrants them.

**Contact methods.** Person has multiple contact methods of different types (landline, cell, email). Model as a `person_contact_method` child table with `type` enum and `value` — not as separate columns on Person.

**Languages.** `preferred_language` is a single nullable varchar on Person. `languages_spoken` is a junction table (`person_language`) — do not use a comma-separated column.

**Emergency contacts.** Child table `person_emergency_contact` with `person_id` FK — not embedded on Person.

**Accessibility needs.** Three boolean columns on Person (`accessibility_hearing`, `accessibility_vision`, `accessibility_mobility`) plus a `mobility_aid` enum (walker / cane / wheelchair) — simple and queryable.

**Deferred to schema phase.** Blob/file storage strategy for OnboardingEvidence documents; external transaction reference format for DuesEvidence; LGL sync mechanics; CE Playwright bridge (temporary, should not shape the schema).

