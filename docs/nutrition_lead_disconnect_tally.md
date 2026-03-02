# Nutrition lead disconnect – tally deliverable

**Reference date:** 2025-03-07 (end of Friday)

**Logic:**
- **Primary memberships only** — `primary_membership_id IS NULL` (secondary memberships excluded).
- **First membership only per member** — for each (coach, member) we count only the membership with the **earliest** `start_date` (renewals / later memberships are ignored).
- **12 weeks from that membership’s start date** — weeks left = min(12, max(0, 12 − weeks_elapsed)), where weeks_elapsed = floor((ref_date − start_date) / 7).
- Total hours = sum(weeks_left) × 0.075.

---

## Summary: supplementary hours to add (one lump sum per coach)

| Coach            | Staff ID                               | Members (first only) | Total weeks | Total hours to add |
|------------------|----------------------------------------|----------------------|-------------|--------------------|
| **Alexandra Wraith** | `2526d373-c09b-424b-9bc8-13d3625186ee` | 16                   | 192         | **14.4**           |
| **James Deacy**      | `277cb159-9cd9-482f-97d2-a58315b964a2` | 134                  | 1,604       | **120.3**          |

Add **14.4 hours** supplementary for Alexandra Wraith and **120.3 hours** for James Deacy in the week you choose (one lump sum each).

---

## How to regenerate the list and totals

Run the following in Supabase SQL (MCP or dashboard).

**Member-level list** (first membership only per member; coach, member, start_date, weeks_elapsed, weeks_left):

```sql
WITH ref AS (SELECT '2025-03-07'::date AS ref_date),
     coaches AS (
       SELECT id AS staff_id, coach_name FROM staff_database
       WHERE id IN ('277cb159-9cd9-482f-97d2-a58315b964a2', '2526d373-c09b-424b-9bc8-13d3625186ee')
     ),
     first_membership AS (
       SELECT DISTINCT ON (c.staff_id, mm.member_id)
         c.coach_name,
         c.staff_id,
         mm.member_id,
         mm.member_name,
         mm.start_date,
         (SELECT ref_date FROM ref) AS ref_date,
         FLOOR(((SELECT ref_date FROM ref) - mm.start_date) / 7.0)::int AS weeks_elapsed,
         LEAST(12, GREATEST(0, 12 - FLOOR(((SELECT ref_date FROM ref) - mm.start_date) / 7.0)::int)) AS weeks_left
       FROM member_memberships mm
       JOIN coaches c ON c.staff_id = mm.nutrition_lead
       CROSS JOIN ref
       WHERE mm.primary_membership_id IS NULL
         AND mm.end_date > (SELECT ref_date FROM ref)
       ORDER BY c.staff_id, mm.member_id, mm.start_date ASC
     )
SELECT coach_name, staff_id, member_id, member_name, start_date, ref_date, weeks_elapsed, weeks_left
FROM first_membership
ORDER BY coach_name, member_name;
```

**Totals per coach** (first membership only per member):

```sql
WITH ref AS (SELECT '2025-03-07'::date AS ref_date),
     coaches AS (
       SELECT id AS staff_id, coach_name FROM staff_database
       WHERE id IN ('277cb159-9cd9-482f-97d2-a58315b964a2', '2526d373-c09b-424b-9bc8-13d3625186ee')
     ),
     first_membership AS (
       SELECT DISTINCT ON (c.staff_id, mm.member_id)
         c.coach_name,
         c.staff_id,
         LEAST(12, GREATEST(0, 12 - FLOOR(((SELECT ref_date FROM ref) - mm.start_date) / 7.0)::int)) AS weeks_left
       FROM member_memberships mm
       JOIN coaches c ON c.staff_id = mm.nutrition_lead
       CROSS JOIN ref
       WHERE mm.primary_membership_id IS NULL
         AND mm.end_date > (SELECT ref_date FROM ref)
       ORDER BY c.staff_id, mm.member_id, mm.start_date ASC
     ),
     agg AS (
       SELECT coach_name, staff_id,
              COUNT(*) AS member_count,
              SUM(weeks_left) AS total_weeks,
              SUM(weeks_left) * 0.075 AS total_hours
       FROM first_membership
       GROUP BY coach_name, staff_id
     )
SELECT coach_name, staff_id, member_count, total_weeks, ROUND(total_hours::numeric, 4) AS total_hours
FROM agg
ORDER BY coach_name;
```

---

## Part 2 (config change) – applied

Migration `set_nutrition_lead_allocation_to_zero` has been applied. Nutrition lead allocation is now 0.

For reference, the change was:

```sql
UPDATE work_estimations
SET per_client_allocation = 0
WHERE 'nutrition_lead' = ANY(staff_roles);
```
