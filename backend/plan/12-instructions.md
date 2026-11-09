# 12: Instructions (Instruksi)

**Purpose:** The DANRU (or an admin) assigns tasks to people across teams and projects, and tracks read status and feedback.
**Manual refs:** §2.8, §3.2.4, Gambar 8, Tabel 8 (Instruksi rows).
**Needs:** 03, 04, 05, 06.
**Contract operations:** `POST /instructions`, `GET /instructions/inbox`, `GET /instructions/sent`, `POST /instructions/{id}/ack` (read + feedback in one call). **Contract changes:** `target: { project_ids, team_ids }` next to `recipient_ids` on `InstructionCreate` (the manual sends to several teams and projects; the web spec's single-target enum loses that), `media_ids`, and `GET /instructions/{id}`.
**Error reasons:**
- `INSTRUCTION_NOT_FOUND`
- `INSTRUCTION_NO_RECIPIENTS`
- `INSTRUCTION_TOO_MANY_RECIPIENTS`
- `INSTRUCTION_RANGE_INVALID`

---

## Phase 12.1: Create and targeting

**Goal:** A DANRU sends an instruction to any mix of projects, teams and individuals within their scope.

**Build**
- **Table `instructions`** (tenant table, RLS):
  - `id`, `organization_id`, `sender_personnel_id`, `instruction_category_id` (INSTRUCTION kind, which carries the urgency)
  - `subject`, `body` (text)
  - `start_date`, `end_date` (DATE)
  - `requires_feedback`
  - timestamps, `deleted_at`
- **Table `instruction_recipients`** (tenant table, RLS):
  - `id`, `organization_id`, `instruction_id`, `personnel_id`, `team_id`, `read_at`, `feedback` (text, nullable), `feedback_at`
  - UNIQUE (`organization_id`, `instruction_id`, `personnel_id`); index (`organization_id`, `personnel_id`, `instruction_id`)
- **`POST /instructions`** (`instruction.create`; idempotent)
  - Body: `{ category_id, subject, body, start_date, end_date, requires_feedback, recipient_ids?, target?: { project_ids?, team_ids? }, media_ids }`.
  - `end_date < start_date` → `INSTRUCTION_RANGE_INVALID`.
  - ⚠ Resolve recipients in `resolveRecipients` (a pure function over loaded data). Comment each step:
    1. Expand `project_ids` into all active teams of those projects.
    2. Union the result with `team_ids`.
    3. Every resulting team must be in scope, otherwise `PERMISSION_DENIED`.
    4. Take all members of those teams as of today.
    5. Add `recipient_ids`. Each must be a member of an in-scope team, and that membership sets their `team_id`.
    6. Remove the sender and dedupe.
    7. No recipients → `INSTRUCTION_NO_RECIPIENTS`. More than 1000 → `INSTRUCTION_TOO_MANY_RECIPIENTS`.
  - Insert the recipients as a **snapshot** with sqlc `:copyfrom`. Comment why a snapshot: later membership changes must not alter who received the instruction.
  - Attach media with purpose `INSTRUCTION`.
- **After commit:** notify every recipient with `INSTRUCTION_RECEIVED`.

**Tests**
- **Unit** (`resolveRecipients`, table-driven): project expansion, dedupe, sender exclusion, an out-of-scope team.
- **HTTP:**
  - A DANRU sends to 2 teams, and every member gets a notification.
  - A member gets 403; a project outside the DANRU's scope gets 403.
  - An empty target fails.
  - Cross-tenant: a team id from another organization is out of scope.

---

## Phase 12.2: Inbox, sent, read and feedback

**Goal:** Recipients read and reply; senders track status (§2.8.1–2.8.6, §3.2.4.2–3.2.4.5).

**Build**
- **Migration:** `pg_trgm` GIN indexes on `instructions.subject` and `instructions.body`. Search works the same way as in Phase 11.2, including sender name.
- **`GET /instructions/inbox` + `ListQuery`:**
  - `q` searches subject, body and sender name (the manual's search by sender; restored).
  - ⚠ The `date`/`month` filter (missing from the mobile spec; the manual has it) matches instructions whose `[start_date, end_date]` **overlaps** the filter range: `start_date <= :to AND end_date >= :from`. Comment the overlap condition.
  - Filters: `category_id`, `status` (`unread`, `needs_feedback`, `done`).
  - Each item includes the category (with urgency), `read_at`, and `needs_feedback` (`requires_feedback AND feedback_at IS NULL`).
- **`GET /instructions/sent` + `ListQuery`** (sender only). Each item includes `{ recipient_count, read_count, feedback_count }`, computed with one `GROUP BY` query.
- **`GET /instructions/{id}`** (contract change; sender or recipient, anyone else gets `NOT_FOUND`):
  - Both see the detail and media views.
  - The **sender** also sees the recipient list with read and feedback status, and every feedback.
  - A **recipient** sees only their own feedback.
- **`POST /instructions/{id}/ack`** `{ read: true, feedback? }` (recipient; idempotent):
  - Sets `read_at = coalesce(read_at, now())`; with `feedback`, also sets `feedback` and `feedback_at` (a second feedback replaces the first and is audited).
  - After commit, when feedback was given, sends `INSTRUCTION_FEEDBACK` to the sender (§3.2.4.5: "DANRU akan mendapatkan … notifikasi membalas instruksi"; the mobile spec missed it).

**Tests**
- **HTTP:**
  - `needs_feedback` shows in the inbox and clears after feedback.
  - Sent counts update.
  - A non-recipient gets 404; a recipient cannot see other recipients' feedback.
  - The date filter matches a multi-day instruction by overlap.
  - Feedback notifies the sender once.
