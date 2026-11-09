# 06: Master data

**Purpose:** Reference data that the feature forms use:
- report and instruction categories
- panic types and stakeholder types (emergency contacts)
- letterheads (Kop Surat) for exports
- external app links
- (leave categories moved to SiAPP's own module 07.5)

**Manual refs:** §2.4 (E-Koperasi Janusa, IKI), §2.7.6, §2.8.6, §2.12.1, §3.2.2.4, §3.2.8.1 (Telp Stakeholder).
**Needs:** 02.
**Contract operations:**
- Admin: `/admin/report-categories`, `/admin/instruction-categories`, `/admin/panic-types`, `/admin/stakeholder-types`, `/admin/letterheads` (list, create, get, patch, delete).
- Field: `GET /report-categories`.
- **Contract changes:** field reads `GET /instruction-categories`, `GET /panic-types`, `GET /projects/{id}/emergency-contacts`, `GET /app-links`, each with `since` for delta sync; admin `/admin/app-links`.
- Personel-owned under D5, not built: `/admin/qualifications`, `/admin/client-types`, `/admin/client-groups`, `/admin/positions`.
**Error reasons:** `CATEGORY_NOT_FOUND`, `CATEGORY_KIND_MISMATCH`, `CATEGORY_NAME_TAKEN`, `PANIC_TYPE_NOT_FOUND`, `STAKEHOLDER_TYPE_NOT_FOUND`, `MASTER_IN_USE`.

**Rule for all masters** (`03-master-data-config.md`, DL-09): a master row that is referenced is **deactivated, not deleted** (`DELETE` on a referenced row → `CONFLICT` with reason `MASTER_IN_USE`; the client offers "non-aktifkan" instead).

---

## Phase 06.1: Categories and panic types

**Goal:** Org admins manage categories and panic types. The app caches them for offline use.

**Build**
- **Table `categories`** (tenant table, RLS)
  - `id`, `organization_id`
  - `kind` ENUM(`REPORT`,`INSTRUCTION`)
  - `name`
  - `urgency` ENUM(`LOW`,`NORMAL`,`HIGH`), nullable
  - `business_lines` (array of the business-line enum; empty = all), so cleaning can have its own categories
  - `sort_order`, `active`, timestamps
  - Partial UNIQUE (`organization_id`, `kind`, `name`) `WHERE active`.
  - `urgency` is required when `kind = INSTRUCTION` and rejected otherwise. Enforce it in the usecase, since manual §2.8 has "high urgent", and add a CHECK constraint as a backstop.
  - ⚠ Comment why one table serves two contract resources: same shape, one set of rules; the handlers fix `kind` from the path.
- **Table `panic_types`** (tenant table, RLS)
  - `id`, `organization_id`, `code`, `name`
  - `stakeholder_type_id` (FK, nullable until 06.2 exists; add the FK in 06.2's migration)
  - `sort_order`, `active`, timestamps
  - UNIQUE (`organization_id`, `code`)
- **Seed per organization** (`masterdata.EnsureDefaults(ctx, orgID)`, called on first sight of an organization like `access.EnsureDefaults`; insert-if-missing only)
  - Panic types: `SECURITY` Keamanan, `MEDICAL` Medis, `FIRE` Kebakaran, `OTHER` Lainnya (the manual's "Lainnya" lets the client add more).
  - Report categories: Penemuan Barang, Kejadian, Kehilangan, Lainnya.
  - Instruction categories: Umum (NORMAL), Pengecekan (HIGH).
- **Admin endpoints**: the contract's CRUD for report categories, instruction categories and panic types. `ORG_ADMIN` only.
- **Field reads** with delta sync: `GET /report-categories`, `GET /instruction-categories`, `GET /panic-types`.
  - ⚠ With `since`, return every row updated after that time **including deactivated ones**, as tombstones in `deleted[]`, so the offline cache can drop them. Without it, return active rows only. Comment this contract (DL-57).
  - Categories are filtered by the business lines of the caller's projects.
- **`masterdata.Service.AssertCategory(ctx, id, kind)`** (exported): `CATEGORY_NOT_FOUND`, or `CATEGORY_KIND_MISMATCH` when the kind is wrong.
- **Leave categories** are not built here: SiAPP owns them, with their quota and balance, in Phase 07.5 (they were a Personel proxy before the leave decision, D2).

**Tests**
- HTTP
  - A new organization gets the defaults; a second call adds nothing.
  - Urgency rules are enforced.
  - A delta sync includes a deactivated row as a tombstone.
  - Deleting a category used by a report returns `MASTER_IN_USE` (test added in 11.1).
  - `AssertCategory` fails for the wrong kind and for another organization's category.

---

## Phase 06.2: Stakeholders, emergency contacts, letterheads and app links

**Goal:** Emergency phone numbers for the panic flow, letterheads for exports, and home-screen links to external apps.

**Build**
- **Table `stakeholder_types`** (tenant table, RLS): `id`, `organization_id`, `code`, `name` (Polisi, Rumah Sakit, Pemadam Kebakaran, …), `default_phone` (E.164, nullable), `sort_order`, `active`, timestamps (DL-14). Seed POLICE, HOSPITAL, FIRE_DEPARTMENT with no default phone.
- **Emergency contacts per project** live in `projects.emergency_contacts` (02.1): `[{ stakeholder_type_id, name, phone }]`, validated here (E.164, known type) when an admin patches the project.
- **`masterdata.Service.ContactsForProject(ctx, projectID, preferredTypeID)`** (exported): the project's contacts, falling back to each type's `default_phone`, with the preferred type first. Phase 09.1 uses it.
- **Table `letterheads`** (tenant table, RLS): `id`, `organization_id`, `name`, `header_text`, `footer_text`, `logo_media_id` (nullable), `is_default`, timestamps. Partial UNIQUE on (`organization_id`) `WHERE is_default`. Used by exports (module 14).
- **Table `app_links`** (tenant table, RLS)
  - `id`, `organization_id`, `name`, `icon_key`
  - `android_package` (nullable), `ios_url_scheme` (nullable), `url` (nullable)
  - `sort_order`, `active`
  - At least one of the three targets must be set.
  - Seed "E-Koperasi Janusa" and "IKI" as **inactive** placeholders until the real links are provided (out of scope, DL-16: links only).
- **Admin endpoints**: `/admin/stakeholder-types`, `/admin/letterheads` (contract), `/admin/app-links` (contract change).
- **Field endpoints**: `GET /projects/{id}/emergency-contacts` (project members only), `GET /app-links`.

**Tests**
- HTTP
  - E.164 validation works when patching a project's contacts.
  - A non-member is denied the project's contacts.
  - The preferred type comes first; a type with no project contact falls back to its default phone.
  - Only one default letterhead per organization.
