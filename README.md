# Session balance – how the data flows

A short guide to how **expected sessions**, **actual hours**, and **weekly balance** fit together.

---

## The big picture

```mermaid
flowchart LR
    subgraph IN[" "]
        A[What coaches are expected to do]
        B[What they actually did]
    end
    subgraph OUT[" "]
        C[Weekly balance]
    end
    A --> C
    B --> C
```

**Expectations** (sessions we expect per week) and **actuals** (hours they worked) are combined to produce a **weekly balance** (over or under).

<small>Sources: expectations from log or computed view; actuals from weekly snapshot; balance from `view_coach_session_balance_sep25`.</small>

---

## Where do expectations come from?

Two ways the system knows “how many sessions” a coach is expected to do:

```mermaid
flowchart TB
    subgraph SOURCES["Ways we get expectations"]
        L[Logged expectations<br/><small>coach_session_expectation_log</small>]
        V[Computed from contract & roles<br/><small>view_coach_session_expectations</small>]
    end
    L --> USED[Used for balance]
    V -.-> REF["(reference / planning)"]
    USED --> BAL[Balance view]
```

| Source | What it is | Used for |
|--------|------------|----------|
| **Logged** | A row per coach per week with expected sessions and role hours. | The balance view uses this. |
| **Computed** | Contract hours minus role hours, turned into “contract sessions” for future weeks. | Planning / reference. |

<small>Table: `coach_session_expectation_log`. View: `view_coach_session_expectations` (uses `get_role_hours_for_week`, `system_config`).</small>

---

## Where do actuals come from?

```mermaid
flowchart LR
    subgraph RAW["Raw data"]
        S[Individual sessions<br/><small>coach_session_actual</small>]
    end
    subgraph AGG["Weekly totals"]
        W[Hours per coach per week<br/><small>coach_weekly_hours_snapshot</small>]
    end
    S --> W
    W --> BAL[Balance view]
```

- **Sessions**: each session (date, coach, attended, etc.) is stored as a row.
- **Snapshot**: those are rolled up into **actual hours per coach per week**. The balance view uses this snapshot.

<small>Tables: `coach_session_actual` → `coach_weekly_hours_snapshot`. Balance view reads `coach_weekly_hours_snapshot.actual_hours`.</small>

---

## How is the balance calculated?

The “Sep 25” balance view does three things: get expectations, adjust them for the week, then compare to actuals.

```mermaid
flowchart TB
    subgraph INPUTS["Inputs"]
        E[Expected sessions<br/><small>from coach_session_expectation_log</small>]
        CAL[Working days in week<br/><small>work_calendar</small>]
        LV[Leave days<br/><small>staff_leave_confirmed</small>]
        ACT[Actual hours<br/><small>coach_weekly_hours_snapshot</small>]
    end
    subgraph STEP1["Step 1: Adjust expectation"]
        E --> ADJ[Expected sessions × working days minus leave × session length]
    end
    CAL --> ADJ
    LV --> ADJ
    subgraph STEP2["Step 2: Compare"]
        ADJ --> BAL[Balance = actual hours − expected hours]
    end
    ACT --> BAL
    BAL --> OUT[Per coach, per week]
```

In words:

1. **Adjust expected hours**  
   Take expected sessions for the week, scale by “how many working days minus leave,” then multiply by session length (e.g. 0.8h from `get_perform_hours()`).

2. **Compare**  
   Balance = actual hours (from snapshot) − that adjusted expected hours.

3. **Result**  
   One balance per coach per week (over or under).

<small>View: `view_coach_session_balance_sep25`. Uses `staff_database`, `work_calendar`, `staff_leave_confirmed`, `coach_weekly_hours_snapshot`, `get_perform_hours()`.</small>

---

## End-to-end flow (one diagram)

```mermaid
flowchart LR
    subgraph EXPECT["Expectations"]
        LOG[Logged per week<br/><small>coach_session_expectation_log</small>]
    end
    subgraph ADJUST["Adjust for week"]
        CAL[Calendar]
        LEAVE[Leave]
        LOG --> ADJ[Expected hours]
        CAL --> ADJ
        LEAVE --> ADJ
    end
    subgraph ACTUAL["Actuals"]
        SNAP[Weekly hours<br/><small>coach_weekly_hours_snapshot</small>]
    end
    ADJ --> BAL[Balance]
    SNAP --> BAL
    BAL --> RESULT[Over / under per coach per week]
```

**Flow:** Logged expectations + calendar + leave → **expected hours**. Weekly snapshot → **actual hours**. Expected vs actual → **balance**.

<small>Main view: `view_coach_session_balance_sep25`.</small>

---

## Quick reference

| Term | Meaning |
|------|--------|
| **Session expectation** | How many sessions we expect a coach to do that week. |
| **Expected hours (adjusted)** | That expectation converted to hours, adjusted for working days and leave. |
| **Actual hours** | Hours from the weekly snapshot (from session data). |
| **Balance** | Actual hours − expected hours (positive = over, negative = under). |

<small>Key table: `coach_session_expectation_log`. Key view: `view_coach_session_balance_sep25`. Snapshot: `coach_weekly_hours_snapshot`.</small>

---

## What counts toward actual hours worked

Total hours are the sum of these **eight** components. Each is either a fixed allocation per client/report, or comes from supplementary config.

```mermaid
flowchart TB
    subgraph ROLES["Role-based hours"]
        RM[Results manager]
        HO[Handoff coach]
        PC[Programming coach]
        RT[Revenue team]
        RL[Renewal lead]
        NL[Nutrition lead]
        HR[HR direct report]
    end
    subgraph EXTRA["Other"]
        SUPP[Supplementary hours]
    end
    RM --> TOTAL[Total hours]
    HO --> TOTAL
    PC --> TOTAL
    RT --> TOTAL
    RL --> TOTAL
    NL --> TOTAL
    HR --> TOTAL
    SUPP --> TOTAL
```

| Component | How the amount is set | Where it’s stored / calculated |
|-----------|------------------------|-------------------------------|
| **Results manager** | Hours per member (no handoff coach yet) | <small>`work_estimations.per_client_allocation` (role `results_manager`) × member count</small> |
| **Handoff coach** | Hours per member assigned to this coach | <small>`work_estimations.per_client_allocation` (`handoff_coach`) × member count</small> |
| **Programming coach** | Hours per member assigned | <small>`work_estimations.per_client_allocation` (`programming_coach`) × member count</small> |
| **Revenue team** | Hours per member assigned | <small>`work_estimations.per_client_allocation` (`revenue_team`) × member count</small> |
| **Renewal lead** | Hours per member assigned | <small>`work_estimations.per_client_allocation` (`renewal_lead`) × member count</small> |
| **Nutrition lead** | Hours per member assigned | <small>`work_estimations.per_client_allocation` (`nutrition_lead`) × member count</small> |
| **HR direct report** | Hours per direct report | <small>`work_estimations.per_client_allocation` (`human_resources`) × count of staff where `direct_report` = this coach</small> |
| **Supplementary** | Default + any extra hours for the week | <small>`staff_supplementary_default_hours` + `staff_supplementary_additional_hours` (per staff, no single config number)</small> |

<small>Role hours: `get_role_hours_for_week()`, `get_human_resources_hours_for_week()`, `get_staff_sup_hours_for_week()`. Stored in `coach_session_expectation_log` and used in `view_coach_session_expectations`.</small>

---

## Config values: amounts and location

These are the main config values that affect session balance and hours. **Location** is the table (and column or key) where the value lives.

### Role hour allocations (hours per client or per report)

| Role | Amount | Location |
|------|--------|----------|
| Results manager | 0.33 h per member | <small>`work_estimations` — `staff_roles` = `results_manager`, `per_client_allocation`</small> |
| Handoff coach | 0.33 h per member | <small>`work_estimations` — `staff_roles` = `handoff_coach`, `per_client_allocation`</small> |
| Programming coach | 0.04 h per member | <small>`work_estimations` — `staff_roles` = `programming_coach`, `per_client_allocation`</small> |
| Revenue team | 0.02 h per member | <small>`work_estimations` — `staff_roles` = `revenue_team`, `per_client_allocation`</small> |
| Renewal lead | 0.06 h per member | <small>`work_estimations` — `staff_roles` = `renewal_lead`, `per_client_allocation`</small> |
| Nutrition lead | 0.075 h per member | <small>`work_estimations` — `staff_roles` = `nutrition_lead`, `per_client_allocation`</small> |
| HR direct report | 1 h per report | <small>`work_estimations` — `staff_roles` = `human_resources`, `per_client_allocation`</small> |

### Contract and session duration (system_config)

| Purpose | Value | Location |
|---------|--------|----------|
| Full-time weekly hours | 38 | <small>`system_config` — `config_key` = `Full Time Hours`, `match_pattern` = `FTE`, `config_value`</small> |
| Part-time weekly hours | 19 | <small>`system_config` — `config_key` = `Part Time Hours`, `match_pattern` = `PTE`, `config_value`</small> |
| Session length (perform) | 0.7 | <small>`system_config` — `config_key` = `duration_perform`, `match_pattern` = `PERFORM`, `config_value`</small> |
| Session length (boxing) | 0.7 | <small>`system_config` — `config_key` = `duration_boxing`, `match_pattern` = `BOX`</small> |
| Session length (squad) | 1.0 | <small>`system_config` — `config_key` = `duration_squad`, `match_pattern` = `SQUAD`</small> |
| Session length (VO2) | 0.7 | <small>`system_config` — `config_key` = `duration_vo2`, `match_pattern` = `VO2`</small> |
| Session length (WAM) | 1 | <small>`system_config` — `config_key` = `duration_wam`, `match_pattern` = `WAM`</small> |

### Other config (system_config)

| Purpose | Value | Location |
|---------|--------|----------|
| Avg perform consumption | 0.4 | <small>`system_config` — `config_key` = `avg_perform_consumption`</small> |
| LC buffer | 0.9 | <small>`system_config` — `config_key` = `lc_buffer`</small> |

<small>Session length for balance is read via `get_perform_hours()` from `system_config` (`duration_perform` or PERFORM match).</small>

