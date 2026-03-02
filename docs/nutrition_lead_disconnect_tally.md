# Nutrition lead disconnect – tally deliverable

**Reference date:** 2025-03-07 (end of Friday)

**Logic:**
- **Primary memberships only** — `primary_membership_id IS NULL` (secondary memberships excluded).
- **First membership only per member** — for each (coach, member) we count only the membership with the **earliest** `start_date` (renewals / later memberships are ignored).
- **New clients only** — include only members with **weeks_left > 0**: those still in their first 12 weeks from membership start, or with a **future** start date (not yet started). Members past their 12-week mark are excluded.
- **12 weeks from that membership’s start date** — weeks left = min(12, max(0, 12 − weeks_elapsed)), where weeks_elapsed = floor((ref_date − start_date) / 7).
- Total hours = sum(weeks_left) × 0.075.

---

## Summary: supplementary hours to add (one lump sum per coach)

| Coach            | Staff ID                               | Members (new only) | Total weeks | Total hours to add |
|------------------|----------------------------------------|--------------------|-------------|--------------------|
| **Alexandra Wraith** | `2526d373-c09b-424b-9bc8-13d3625186ee` | 16                 | 192         | **14.4**           |
| **James Deacy**      | `277cb159-9cd9-482f-97d2-a58315b964a2` | 134                | 1,604       | **120.3**          |

Add **14.4 hours** supplementary for Alexandra Wraith and **120.3 hours** for James Deacy in the week you choose (one lump sum each).

---

## Member names (new clients only: first 12 weeks or future start)

**Alexandra Wraith (16 members):**  
Antony Pearce, Belinda Tumbers, Daniel Cawthorne, Drew Riethmuller, Emmanuel Armand, Gareth Bryant, James Nihill, James Zipeure, Mario Raciti, Nina Zhang, Rachael Rodgers, Reece Corbett-Wilkins, Robert Scappatura, Rod Smith, Taylor Spensieri, Tribeni Lodh.

**James Deacy (134 members):**  
Aaron Hendershot, Adam Hirst, Aileen Sang, Alan Hunter, Alex Badran, Allie Moxon, Andrew Mckillop, Andrew Quinn, Angela Vuksic, Angus Firth, Angus Hamilton, Angus Tse, Anthony Gasparre, Arumugam Yathurshan, Ashutosh Gupta, Asli Cetin, Bernadette Sukkar, Blair Nicholls, Blake Single, Brian Mulder, Brigid Archibald, Bruce Connors, Candice Joll, Carly Martin, Carola Jonas, Caroline Choi, Carrie Barker, Chantel Blankfield, Chloe Bartlett, Chris Allenby, Chris Gray, Chris Hall, Chris Prestipino, Christopher Fennell, Christos Siafakas, Claire Quigley, Conor Martin, Damien Bailey, David Fam, Dean Hyde, Dimitri Courtelis, Duncan Clubb, Edward Farrell, Elliott Shadforth, Eric Peterson, Eric Yip, Ershad Alkozai, Fernando Kularante, Fiona Bones, Geoffrey Brown, Grant McCarthy, Greg Cameron, Isaac Kutcher, James Blake, James Davie, James Loughhead, Jason Pohl, Jeremy Cabral, Jessica Yue, Joe Caristo, Jordan Brown, Josh Mchutchison, Joshua Rogers, Justin Crawford, Karla Robinson, Kate Cooper, Kathy Xu, Kevin Forder, Kostas Thanos, Kristy Kerswell, Lauren Zusy, Lixian Liang, Luci Ellis, Luke O'kane, Magdeline Baramalis, Maggie Kolasky, Marel Pencev, Maria Clarkin, Mathew Camilleri, Matt Congiusta, Matthew Bauld, Matthew Stephens, Matthew Winchur, Matt Williams, Megan Evetts, Melissa Saadat, Melissa Sheehan, Meredith Jordan, Michael Azzi, Michael Miller, Michael Peters, Mike Boutel, Murray Whiteside, Nass Martino, Niamh Scanlon, Nick Perrott, Nigel Bradshaw, Olivia McArdle, Patrick Ibbotson, Paul Bennett, Paul Potter, Paul Slaven, Philip Gartland, Phillip Blackmore, Phillip Silver, Pierre Peyronny, Poonam Naiker, Pranav Bahl, Raymond Roser, Rebecca Larking, Richard Wakefield, Rodrigo Menseses, Rohan Advani, Rosemary Norgate, Rowena Morgan, Sagar Shettar, Sam Mellor, Sandeep Pillay, Sanjeev Prasad, Sarah Tempest, Sebastien Mckenna, Spencer Ivey, Stefan Duro, Stefano Martincigh, Stephen Jani, Stephen Panizza, Tim Johnston, Tim Meurer, Trent Primmer, Vesna Garnett, Victoria Denholm, Vikas Nahar, Vivek Vyas, WIll Hamilton.

---

## How to regenerate the list and totals

Run the following in Supabase SQL (MCP or dashboard).

**Member-level list** (new clients only: first 12 weeks or future start; coach, member, start_date, weeks_elapsed, weeks_left):

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
WHERE weeks_left > 0
ORDER BY coach_name, member_name;
```

**Totals per coach** (new clients only: first 12 weeks or future start):

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
     new_only AS (SELECT * FROM first_membership WHERE weeks_left > 0),
     agg AS (
       SELECT coach_name, staff_id,
              COUNT(*) AS member_count,
              SUM(weeks_left) AS total_weeks,
              SUM(weeks_left) * 0.075 AS total_hours
       FROM new_only
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
