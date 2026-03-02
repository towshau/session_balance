# Nutrition lead disconnect – tally deliverable

**Reference date:** 2025-03-07 (end of Friday)  
**Logic:** For each member where the coach is `nutrition_lead`, weeks left = min(12, max(0, 12 − weeks_elapsed)) from membership start. Total hours = sum(weeks_left) × 0.075.

---

## Summary: supplementary hours to add (one lump sum per coach)

| Coach            | Staff ID                               | Total weeks | Total hours to add |
|------------------|----------------------------------------|-------------|--------------------|
| **Alexandra Wraith** | `2526d373-c09b-424b-9bc8-13d3625186ee` | 192         | **14.4**           |
| **James Deacy**      | `277cb159-9cd9-482f-97d2-a58315b964a2` | 1,748       | **131.1**          |

Add **14.4 hours** supplementary for Alexandra Wraith and **131.1 hours** for James Deacy in the week you choose (one lump sum each).

---

## How to regenerate the list and totals

Run the following in Supabase SQL (MCP or dashboard).

**Member-level list** (coach, member, start_date, weeks_elapsed, weeks_left):

```sql
WITH ref AS (SELECT '2025-03-07'::date AS ref_date),
     coaches AS (
       SELECT id AS staff_id, coach_name FROM staff_database
       WHERE id IN ('277cb159-9cd9-482f-97d2-a58315b964a2', '2526d373-c09b-424b-9bc8-13d3625186ee')
     ),
     members AS (
       SELECT
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
     )
SELECT coach_name, staff_id, member_id, member_name, start_date, ref_date, weeks_elapsed, weeks_left
FROM members
ORDER BY coach_name, member_name;
```

**Totals per coach**:

```sql
WITH ref AS (SELECT '2025-03-07'::date AS ref_date),
     coaches AS (
       SELECT id AS staff_id, coach_name FROM staff_database
       WHERE id IN ('277cb159-9cd9-482f-97d2-a58315b964a2', '2526d373-c09b-424b-9bc8-13d3625186ee')
     ),
     members AS (
       SELECT
         c.coach_name,
         c.staff_id,
         LEAST(12, GREATEST(0, 12 - FLOOR(((SELECT ref_date FROM ref) - mm.start_date) / 7.0)::int)) AS weeks_left
       FROM member_memberships mm
       JOIN coaches c ON c.staff_id = mm.nutrition_lead
       CROSS JOIN ref
       WHERE mm.primary_membership_id IS NULL
         AND mm.end_date > (SELECT ref_date FROM ref)
     ),
     agg AS (
       SELECT coach_name, staff_id,
              SUM(weeks_left) AS total_weeks,
              SUM(weeks_left) * 0.075 AS total_hours
       FROM members
       GROUP BY coach_name, staff_id
     )
SELECT coach_name, staff_id, total_weeks, ROUND(total_hours::numeric, 4) AS total_hours
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
