# Village Green — Ontology Summary
*Working document — session summary as of May 2026*

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
- Languages spoken (other than English) ← volunteer application captures this; worth carrying at Person level
- Street address
- Contact methods: landline, cell, email (typed, multiple)
- Accessibility needs: hearing, vision, mobility (walker/cane/wheelchair) — modeled at Person level, not Member, because operationally relevant for volunteers too
- Emergency contact(s): name, phones, email, relationship

**Notes:**
- A Person who is only a User may have null address and other fields — acceptable, meaningful nulls
- Aligns with LGL Constituent; `lgl_constituent_id` should be carried for sync purposes

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
- Driving credentials validated flag (set at graduation from PendingVolunteer; documents archived on PendingVolunteer)

---

### Household *(thin grouping entity)*
Links two Members sharing a dwelling and a dues obligation. Dual Household is a billing convenience, not an ontological unit — two Persons, two Members, one dues arrangement.

**Properties:**
- Reference to Member 1, Member 2
- Dues arrangement (level, payment method, frequency)

**Scope:** Appears only in membership dues context. No role in service requests, communications, or events.

---

### PendingMember *(transient, pre-role)*
A Person who has expressed interest in membership and is undergoing assessment. Not yet a Member. Exists until graduation (→ Member) or rejection.

**Properties:**
- Application date
- Membership Ambassador (reference to Volunteer or staff User)
- Household type: Single or Dual
- Ambassador notes (free text — currently the only record of the subjective suitability assessment)
- Waiver signed flag, signature date
- Dues level selected, payment method selected
- Graduation event reference (links to resulting Member record)

**Notes:**
- The Ambassador visit is the current assessment gate; criteria are informal and not tracked structurally
- Policy in flux: board considering open membership without dues gate; service eligibility assessment may become a distinct step
- Most contact/personal info collected here graduates to Person on approval

---

### PendingVolunteer *(transient, pre-role)*
A Person who has submitted a volunteer application and is progressing through onboarding. Not yet a Volunteer. Exists until graduation (→ Volunteer) or withdrawal.

**Onboarding is not a fixed linear pipeline** — it is a dynamic checklist of RequirementItems, shaped by the service capabilities the applicant has selected. Checklist completion drives progression.

**Properties:**
- Application date
- Volunteer Ambassador (reference)
- Service capabilities expressed interest in (determines which RequirementItems are generated)
- Waiver signed flag (confidentiality + liability), signature date
- Background check: authorization date, completion date, outcome
- Orientation: date, format (Zoom/in-person)
- Badge: photo received, lanyard preference (standard blue / rainbow Circle of Pride)
- Archived driver documents (if Driver capability selected): license copy, vehicle registration(s), insurance proof — preserved after graduation as evidence of validation, not actively managed
- Vehicle records (if Driver capability selected): model, make, year, plate, state, insurance company, license restrictions
- Ambassador/staff notes

---

### OnboardingRequirement *(owned by PendingVolunteer)*
A single checklist item required for a specific volunteer capability. Generated dynamically based on capability selections.

**Properties:**
- Requirement type (e.g. background check authorization, driver's license copy, vehicle registration, insurance proof, orientation attendance)
- Triggering capability (e.g. Driver–Rides, Village Friends)
- Status: outstanding / received / waived
- Received date

---

## Key Design Decisions

1. **Roles pattern:** Person is the person; roles (User, Member, Volunteer) are separate entities keyed by pId. Role membership is implicit in the existence of a role record, not stored as an array on Person.

2. **Address on Person:** Address is a fact about the person, not the role. Nulls for User-only persons are acceptable and meaningful.

3. **Accessibility needs on Person:** Not Member-only — operationally relevant for volunteers too. Designed for current state but correct at the person level.

4. **Status history owned by VG:** VG is the authority for Member and Volunteer role history (active/inactive periods with dates). LGL may be informed but does not own this.

5. **Pending as pre-role:** PendingMember and PendingVolunteer are distinct transient entities, not states on Member/Volunteer. A Person is not a Member or Volunteer until graduation.

6. **Driver as capability bundle, not sub-role:** Driving credentials are validated once at onboarding (per current state law requirement) and archived. No ongoing credential lifecycle management. Driver is a set of capabilities on Volunteer with archived onboarding documentation.

7. **Household is thin:** Exists solely to express shared dues between two Members. Does not extend into service, communications, or events.

8. **Dynamic onboarding checklist:** PendingVolunteer onboarding is modeled as a set of OnboardingRequirements derived from capability selections, not a fixed pipeline. Supports the back-and-forth reentry reality of current process.

---

## Entities Not In Scope (Current Phase)

- Organizations / non-natural-person persons
- Events, registration, attendance
- Service requests and volunteer pairing (next major domain after onboarding)
- Financial transactions, dues payment processing (LGL/external responsibility)
- Detailed communications history (LGL responsibility)

---

## Open Questions

- Will service eligibility assessment become a formal step in PendingMember onboarding as board policy evolves?
- Should languages spoken be on Person, or is it Volunteer-specific (for service matching)?
- Emergency contacts: modeled as structured sub-entities on Person, or as light free-text on the pending records only?

