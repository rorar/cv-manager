# Volunteer Work Migration: `volunteer_work` to Custom Sections

## Overview

PR #101 introduced a dedicated `volunteer_work` table. This migration re-implements volunteer work using the existing `custom_sections` + `custom_section_items` tables, eliminating the need for a separate `volunteer_work` table. All existing volunteer data is preserved with a one-time, idempotent migration at server startup.

---

## 1. Migration Strategy

### Detection

At startup (inside the `if (!PUBLIC_ONLY)` block, after table creation and before default data insertion), the migration runs conditionally:

```js
// Step 2i: Migration - volunteer_work → custom_sections
try {
    const alreadyMigrated = db.prepare("SELECT value FROM settings WHERE key = 'migration_volunteer_to_custom_section'").get();
    const hasVolunteerData = db.prepare("SELECT COUNT(*) as cnt FROM volunteer_work").get();

    if (!alreadyMigrated && hasVolunteerData.cnt > 0) {
        // Run migration
        // ...
        db.prepare("INSERT OR REPLACE INTO settings (key, value) VALUES ('migration_volunteer_to_custom_section', 'done')").run();
    }
} catch (err) { console.log('Migration check (volunteer to custom sections):', err.message); }
```

**Idempotency**: The `settings` table flag prevents re-running. Re-running on a fresh install with zero volunteer rows is a no-op.

### Steps

1. **Check** — Query `settings` for `migration_volunteer_to_custom_section=done`. If set, skip entirely.
2. **Check** — `SELECT COUNT(*) FROM volunteer_work`. If 0 rows, skip (nothing to migrate).
3. **Create custom section**:
   ```sql
   INSERT INTO custom_sections (name, section_key, layout_type, icon, sort_order, visible, metadata)
   VALUES (
     'Volunteer Work',
     'volunteer_work',
     'timeline',
     'volunteer_activism',
     600,  -- after built-in sections
     1,
     '{"show_on_timeline": true}'
   );
   ```
   `section_key = 'volunteer_work'` distinguishes this as the migrated volunteer section. `layout_type = 'timeline'` aligns with the existing `buildTimelineItems()` logic that reads custom sections with `layout_type = 'timeline'` and `metadata.show_on_timeline = true`.
4. **Migrate each row** — For every `volunteer_work` row, insert one `custom_section_item`:
   ```sql
   INSERT INTO custom_section_items
     (section_id, title, subtitle, description, metadata, sort_order, visible)
   VALUES
     (?, ?, ?, ?, ?, ?, ?)
   ```
   - `section_id` = the new custom section's id
   - `title` = `null` (roles are not a single title; each role is a timeline sub-entry)
   - `subtitle` = `organization` from the source row (acts as the group header in timeline rendering)
   - `description` = `description` from the source row
   - `metadata` = JSON object (see Section 2)
   - `sort_order` = `volunteer_work.sort_order`
   - `visible` = `volunteer_work.visible`
5. **Insert migration flag**:
   ```sql
   INSERT OR REPLACE INTO settings (key, value) VALUES ('migration_volunteer_to_custom_section', 'done')
   ```

### `volunteer_work` Table After Migration

The table **is not dropped**. It is left in place (a) for rollback capability during development, and (b) because the existing `gatherCvData()` function and several API routes still reference it. Dropping the table is a separate cleanup task.

---

## 2. Metadata Schema

Each migrated `custom_section_items` row stores volunteer data in the `metadata` column as a JSON object:

```json
{
  "start_date": "2020-01",
  "end_date": null,
  "country_code": "",
  "summary": "...",
  "roles": [
    { "title": "Team Lead", "start_date": "2022-07", "end_date": null },
    { "title": "Volunteer", "start_date": "2020-01", "end_date": "2022-06" }
  ]
}
```

| Field | Type | Description |
|---|---|---|
| `start_date` | `string \| null` | ISO YYYY-MM or null. Earliest role start date. Used for sorting and `formatPeriod()` in the timeline. |
| `end_date` | `string \| null` | ISO YYYY-MM or null. Latest role end_date (null means "present"). |
| `country_code` | `string` | Reserved for future use (country flag in timeline). Currently always `""`. |
| `summary` | `string \| null` | The organization-level description from `volunteer_work.description`. |
| `roles` | `array` | Original roles array from `volunteer_work.roles`, preserved verbatim. Each role is `{title, start_date, end_date}`. |

**Constraints**:
- `roles` is required and must be a non-empty array. Even a single-role organization has a roles array with one element.
- Dates are ISO `YYYY-MM` or `YYYY` format, consistent with all other date fields in the system.
- `start_date` is the min of all role start_dates; `end_date` is the max of all role end_dates.

---

## 3. Indexing

**No new indexes are required.**

The existing index `idx_custom_section_items_visible_sort ON custom_section_items(visible, sort_order)` (created at startup) covers all query patterns for the migrated data:

| Query | Covered by |
|---|---|
| `WHERE visible = 1 ORDER BY sort_order ASC` (public timeline items) | `idx_custom_section_items_visible_sort` |
| `WHERE section_id = ? ORDER BY sort_order ASC` (admin items list) | `idx_custom_section_items_visible_sort` (covering leftmost prefix `visible, sort_order`; `section_id` filter is applied post-index-scan) |

The `section_key = 'volunteer_work'` lookup uses the `UNIQUE` constraint on `custom_sections.section_key` — no index needed.

**Index on `custom_sections.layout_type`** would be useful if many custom sections existed, but with only one or a few timeline-type sections, a table scan is negligible.

---

## 4. Query Patterns and Role Expansion in `buildTimelineItems()`

### Current Behavior (pre-migration)

`buildTimelineItems()` reads `volunteer_work` directly and **expands** each role into a separate timeline entry:

```js
// lines 856-877 of server.js
const volunteerRaw = db.prepare(
    publicView
        ? 'SELECT id, organization, roles FROM volunteer_work WHERE visible = 1 ORDER BY sort_order ASC'
        : 'SELECT id, organization, roles, visible FROM volunteer_work ORDER BY sort_order ASC'
).all();
volunteerRaw.forEach(v => {
    const roles = JSON_SAFE_PARSE(v.roles);
    roles.forEach((role, ridx) => {
        volunteer.push({
            id: `vol_${v.id}_${ridx}`,          // composite id: vol_{row_id}_{role_index}
            company: v.organization,
            role: role.title || '',
            period: formatPeriod(role.start_date, role.end_date),
            start_date: role.start_date,
            end_date: role.end_date,
            countryCode: '',
            visible: publicView ? true : !!v.visible,
            logo: null,
            sort_order: (v.sort_order || 0) * 100 + ridx  // interleave roles within same org
        });
    });
});
```

Each role gets its own timeline card. The `sort_order` formula `v.sort_order * 100 + ridx` ensures all roles of one organization are adjacent in the sorted order.

### Post-Migration Behavior

After migration, the volunteer section becomes a `timeline`-layout custom section. The `buildTimelineItems()` loop for custom timeline items already handles the correct shape:

```js
// lines 880-905 of server.js
const timelineSections = db.prepare(
    `SELECT id, metadata FROM custom_sections WHERE layout_type = 'timeline' AND visible = 1`
).all().filter(section => {
    const meta = section.metadata ? JSON.parse(section.metadata) : {};
    return meta.show_on_timeline;
});
const customItems = [];
for (const section of timelineSections) {
    const items = db.prepare(
        publicView
            ? 'SELECT * FROM custom_section_items WHERE section_id = ? AND visible = 1 ORDER BY sort_order ASC'
            : 'SELECT * FROM custom_section_items WHERE section_id = ? ORDER BY sort_order ASC'
    ).all(section.id);
    for (const item of items) {
        const meta = item.metadata ? JSON.parse(item.metadata) : {};
        customItems.push({
            id: `cs_${item.id}`,
            company: item.subtitle || '',          // subtitle = organization
            role: item.title || '',                // title = null (roles are in metadata)
            period: formatPeriod(meta.start_date, meta.end_date),
            start_date: meta.start_date || '',
            end_date: meta.end_date || '',
            countryCode: meta.country_code || '',
            visible: publicView ? true : !!item.visible,
            logo: item.image || null
        });
    }
}
```

**The issue**: Currently, one `custom_section_items` row maps to one timeline card. After migration, each organization (one `custom_section_item`) has multiple roles but produces only one timeline card — the role expansion logic present in the `volunteer_work` branch is absent for the custom items branch.

**Required change**: In `buildTimelineItems()`, when processing custom items whose metadata contains a `roles` array, expand each role into a separate timeline entry (analogous to the current `volunteer_work` expansion):

```js
// In the custom items loop:
const roles = meta.roles;
if (roles && Array.isArray(roles)) {
    roles.forEach((role, ridx) => {
        customItems.push({
            id: `cs_${item.id}_${ridx}`,
            company: item.subtitle || '',
            role: role.title || '',
            period: formatPeriod(role.start_date, role.end_date),
            start_date: role.start_date || '',
            end_date: role.end_date || '',
            countryCode: meta.country_code || '',
            visible: publicView ? true : !!item.visible,
            logo: item.image || null,
            sort_order: (item.sort_order || 0) * 100 + ridx
        });
    });
} else {
    // Non-role custom items (single-entry timeline items)
    customItems.push({
        id: `cs_${item.id}`,
        company: item.subtitle || '',
        role: item.title || '',
        period: formatPeriod(meta.start_date, meta.end_date),
        start_date: meta.start_date || '',
        end_date: meta.end_date || '',
        countryCode: meta.country_code || '',
        visible: publicView ? true : !!item.visible,
        logo: item.image || null,
        sort_order: item.sort_order || 0
    });
}
```

The `roles` field existence acts as a sentinel: if metadata has `roles`, expand; otherwise treat as a single-entry item.

### Final Timeline Merge

```js
return sortTimelineItems([...experiences, ...volunteer, ...customItems]);
```

The combined list is sorted by `sort_order` (which uses `item.sort_order * 100 + ridx` for expanded roles, preserving insertion order).

---

## 5. Data Integrity

### Preservation Guarantees

| Source Field | Destination | Guarantee |
|---|---|---|
| `volunteer_work.id` | Lost (no mapping stored) | Not needed — timeline uses composite `vol_{id}_ridx` IDs which are transient |
| `volunteer_work.organization` | `custom_section_items.subtitle` | Exact string copy |
| `volunteer_work.description` | `custom_section_items.description` | Exact string copy |
| `volunteer_work.roles` (JSON array) | `custom_section_items.metadata.roles` | Exact JSON serialization; parsed and re-serialized to ensure valid JSON |
| `volunteer_work.sort_order` | `custom_section_items.sort_order` | Integer preserved exactly |
| `volunteer_work.visible` | `custom_section_items.visible` | Integer preserved exactly (1/0) |
| Per-role `start_date` | `metadata.start_date` (earliest) | Min() of all role start_dates |
| Per-role `end_date` | `metadata.end_date` (latest) | Max() of all role end_dates |

### Edge Cases

1. **Empty `roles` array**: A `volunteer_work` row with `roles = []` still migrates. The resulting `custom_section_items` row will have `metadata.roles = []` and timeline rendering will show a card with no role entries (summary-only card). This matches pre-migration behavior where the public `/api/volunteer` endpoint would return the empty roles array.

2. **`organization` is `NOT NULL`** in the schema, so subtitle will always be populated.

3. **`roles` stored as JSON string in SQLite**: The migration calls `JSON_SAFE_PARSE(v.roles)` to deserialize, then `JSON.stringify()` to serialize into the new metadata field, ensuring the stored value is valid JSON regardless of the original string's formatting.

4. **Roles with missing `start_date` or `end_date`**: The per-role dates are stored verbatim. `formatPeriod('', '2022-06')` will render as "2022-06" (start is empty), consistent with existing `formatPeriod` behavior.

5. **Sort order collision**: Two `volunteer_work` rows with the same `sort_order` both migrate correctly. The custom items get their original sort orders, and `sortTimelineItems()` resolves ties by insertion order (stable sort).

6. **All visible=0**: Migration still runs; creates invisible custom section items. Admin can toggle visibility later.

7. **Database migration order**: The `volunteer_work → custom_section_items` migration runs before Step 4 (auto-create default dataset), so the dataset snapshot will contain both the legacy `volunteer_work` array and the new `customSections` — this is fine for rollback; both are present during the transition.

---

## Summary

The migration is:
- **Idempotent** — guarded by a `settings` flag
- **Lossless** — every field is preserved exactly
- **No new indexes needed** — existing composite index covers all queries
- **Non-destructive** — `volunteer_work` table remains for reference during transition
- **Consistent with existing patterns** — uses the same `settings` flag pattern as `dates_normalized`, same `layout_type='timeline'` + `show_on_timeline` convention as other timeline custom sections