## VG Technical Architecture — Session Summary (2026-05-22)

### Project Context
Village Green (VG) is an open-source platform to replace Club Express (CE) for TVCRI, a federated network of Villages. Wood River Village is the initial deployment. The platform is designed multi-village from day one, with TVCRI operating shared infrastructure. CE's metro-areas become first-class Villages in VG.

**Current member/volunteer numbers:** 616 members, 454 volunteers, 174 overlap — ~896 unique individuals.

### Stack Decisions
- **Runtime:** Node.js
- **Database:** MySQL 8.x (Azure Database for MySQL in production, local Docker container for dev)
- **Email:** Transactional email service (Postmark or similar) — SendGrid free tier insufficient given fan-out volume
- **Scheduler:** MySQL `CREATE EVENT` — confirmed ON by default in both 8.0 and 8.4 (contrary to some sources including Gemini, which was wrong while citing the correct page)

### Service Request Email Architecture
The service request pipeline is the primary email driver:
- Member in Village A requests Service C
- Notification fans out to all Village B volunteers matching Service C — potentially dozens of emails per request
- Volunteers receive email with a link to a VG confirm route
- First volunteer to click confirms; subsequent visitors see "too late" page
- No automated inbound handling — confirmation happens via link, not email reply
- Second-notice email fires after a configurable timeout with no confirmation
- Follow-up and confirmation emails round out the template set (~5 templates total)

### Component Architecture
- **API** — stateless, sole holder of DB credentials, only component that touches tables
- **Mailer sidecar** — separate container, no DB access, calls API as a client, sole responsibility is sending email
- **MySQL EVENT** — handles state transitions (e.g. flagging `second_notice_pending`) independently of Node processes; survives mailer downtime and ensures DB consistency
- **Mailer polls API** for pending notices; API enforces village-level data boundaries

```yaml
services:
  db:
    image: mysql:8.0
  api:
    build: ./api
  mailer:
    build: ./mailer
    depends_on:
      - db
```

### Multi-Village Architecture
- `village_id` foreign key on all relevant tables from day one — no retrofit later
- Cross-village volunteer confirmations currently allowed in CE (possibly by accident of missing config) — VG will make this a deliberate, configurable policy
- API enforces village-level data boundaries for all consumers
- Mailer is just another API client with its own credentials

### Google Maps (Parked)
Iframe embed via Maps Embed API noted as a future feature for showing routes between addresses. Prototype-friendly no-key URL exists for early testing.

### Claude Chat Memory Note
Chat sessions are independent — no shared context between sessions. Code artifacts, decisions, and file contents from prior sessions are not available unless explicitly pasted in. The memory summaries Anthropic maintains capture themes and relationships, not code. **Recommended practice:** end significant sessions with a summary document; paste into next session opener. Consider a `CONTEXT.md` in the repo mirroring the `MEMORY.md` pattern from Claude Code.

---
