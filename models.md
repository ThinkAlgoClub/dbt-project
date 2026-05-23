## Models Based on the CSV File Contents

### CSV Structure
| Column | Example Value | Issues |
|--------|---------------|--------|
| `student_id` | STU001 | Some students appear multiple times (transfers) |
| `first_name` | Maria | – |
| `last_name` | Garcia | – |
| `school_id` | SCH101 | – |
| `school_name` | IDEA Frontier Academy | – |
| `state` | TX, Texas, Texas (with space), OH, FL | Inconsistent: `'Texas'`, `'Texas '`, `'TX'` |
| `grade_level` | 3, N/A, K | `'N/A'` placeholder; `'K'` for kindergarten |
| `entry_date` | 2025-08-18 or 08-18-2025 | Two formats: YYYY-MM-DD and MM-DD-YYYY |
| `exit_date` | (blank) or 2025-11-03 | Blank = NULL (still enrolled) |
| `entry_code` | E1, T1 | E1 = new, T1 = transfer |
| `exit_code` | W1, W2, W3, T1 | Withdrawal codes |
| `status` | Active, Withdrawn, Transferred | – |

---

### Model 1: `stg_sis_enrollments` (Staging)

**Materialization:** `view` (recalculated each time; no storage)

**Grain:** Exactly one row per raw CSV row (1:1). No deduplication, no aggregation.

**Transformations (applied in order):**

| Action | Column(s) Affected | Rule |
|--------|--------------------|------|
| Generate surrogate key | `enrollment_id` | Hash of `student_id` \|\| `'|'` \|\| `school_id` \|\| `'|'` \|\| `entry_date` (original string) |
| Trim whitespace | `state`, `entry_code`, `exit_code`, `status` | Apply `TRIM()` to remove leading/trailing spaces |
| Standardize state | `state_code` | If `state` equals `'Texas'` or `'Texas '` → `'TX'`; else keep as‑is (must be `'TX'`, `'OH'`, or `'FL'`) |
| Convert grade | `grade_level_int` | `'N/A'` → `NULL`; `'K'` → `0`; else cast to integer |
| Parse entry date | `entry_date_clean` | Try `YYYY-MM-DD` first; if fails, try `MM-DD-YYYY`; if both fail → `NULL` |
| Parse exit date | `exit_date_clean` | Same as entry date; empty string or `NULL` remain `NULL` |
| Preserve codes and status | `entry_code`, `exit_code`, `status` | Already trimmed; pass through |
| Add staging timestamp | `staged_at` | `CURRENT_TIMESTAMP()` |

**Filters:** Exclude rows where `student_id IS NULL` (minimal quality).

---

### Model 2: `dim_students` (Dimension)

**Materialization:** `table` (full refresh daily, or incremental if large)

**Grain:** One row per unique `student_id` (current snapshot only).

**Logic (executed in order):**

1. **Read from** `stg_sis_enrollments`.
2. **Window ranking:** For each `student_id`, order rows by `entry_date_clean` descending (newest first). Assign `rn = 1,2,...`.
3. **Filter:** Keep only rows where `rn = 1` (the most recent enrollment record for the student).
4. **Derive columns** from that selected row:
   - `student_dim_id` – surrogate key: hash of `student_id`
   - `natural_student_id` – same as `student_id` (rename for clarity)
   - `first_name`, `last_name`
   - `current_school_id`, `current_school_name`
   - `current_state` – from `state_code`
   - `current_grade` – from `grade_level_int`
   - `current_enrollment_start_date` – from `entry_date_clean`
   - `current_enrollment_end_date` – from `exit_date_clean`
   - `current_status` – from `status`
   - `current_entry_code`, `current_exit_code`
   - `is_currently_enrolled` – boolean: `TRUE` if `exit_date_clean IS NULL` AND `status = 'Active'`; else `FALSE`
   - `dim_updated_at` – `CURRENT_TIMESTAMP()`

**Handling of transfers:** A student like `STU003` (Aisha) has two rows. The ranking picks the row with later entry date (Zenith Academy). The dimension shows her as currently enrolled at Zenith, with `is_currently_enrolled = TRUE`. The previous enrollment (Quest) is not in the dimension but remains in the fact table.

---

### Model 3: `fct_enrollments` (Fact)

**Materialization:** `incremental` (daily merge using `enrollment_fact_id` as unique key). Partitioned by month on `enrollment_start_date`, clustered by `student_id`.

**Grain:** One row per student per continuous enrollment period at a single school. A transfer creates two rows; a re‑enrollment after a gap creates two rows.

**Transformations (step by step):**

1. **Read from** `stg_sis_enrollments`.
2. **Order within each student** by `entry_date_clean` ascending.
3. **Peek at next start date** using `LEAD(entry_date_clean)` – call it `next_entry_date`.
4. **Adjust exit date for overlaps** (business rule: no two schools at the same time):
   - If `exit_date_clean IS NULL` → keep `NULL` (open enrollment).
   - Else if `exit_date_clean >= next_entry_date` → set `enrollment_end_date = next_entry_date - 1 day`.
   - Else → set `enrollment_end_date = exit_date_clean`.
   - Add `was_end_date_adjusted` flag = `TRUE` when the second case applied.
5. **Derive columns:**
   - `enrollment_fact_id` – hash of `student_id`, `school_id`, `enrollment_start_date`
   - `student_id`, `school_id`, `school_name`, `state_code`
   - `grade_at_enrollment` – from `grade_level_int`
   - `enrollment_start_date` – from `entry_date_clean`
   - `enrollment_end_date` – adjusted value (or NULL)
   - `days_enrolled_to_date` – compute: `DATEDIFF(day, start, COALESCE(end, CURRENT_DATE))`
   - `entry_code`, `exit_code`, `status`
   - `was_end_date_adjusted` (boolean)
   - `is_open_enrollment` – `TRUE` if `enrollment_end_date IS NULL`
   - `fact_updated_at` – `CURRENT_TIMESTAMP()`
6. **Incremental filter** (when not a full refresh):  
   Include rows where `enrollment_start_date` > `(max start date already in table) - 30 days` OR `enrollment_end_date` > `(max end date already in table) - 30 days`. This catches late‑arriving exit dates.

**Example results from the CSV:**

| Student | Raw rows | After adjustment | Rows in fact |
|---------|----------|------------------|--------------|
| STU001 (no transfer) | 1 | no overlap | 1 |
| STU003 (Aisha, transfer) | 2 | no overlap | 2 (Quest, Zenith) |
| STU008 (Liam, overlap) | 2 | adjusted: Rise ends day before Frontier starts | 2 (both kept) |
| STU015 (Mia, transfer) | 2 | no overlap (consecutive dates) | 2 |

---

### Custom Test: No Overlapping Enrollments

**Purpose:** Ensure that after overlap resolution, no student has two enrollment periods at different schools with overlapping dates.

**Test logic (expressed in words):**  
Join `fct_enrollments` to itself on `student_id`, taking two distinct rows (by `enrollment_fact_id`). Check if:
- Row A's start date ≤ Row B's end date (or B has no end date), AND
- Row B's start date ≤ Row A's end date (or A has no end date), AND
- The schools are different.

If any such pair exists, the test fails, indicating an unresolved overlap (or a business rule change requiring dual enrollment).

---

### Stakeholder Question Handling

**Question:** *"How many students are currently enrolled across all Texas campuses?"*

**Ambiguities identified:** time point, definition of "enrolled" (active only? suspended?), scope (physical campuses only? virtual Texas‑headquartered?), uniqueness (distinct students or total enrollments?), exclusions (pre‑K?).

**Resolution:**  
1. Ask the superintendent: clarify each ambiguity.  
2. Document the final definition as a dbt metric with explicit filters (e.g., `campus_type = 'Physical'`, `school_state = 'TX'`, `is_open_enrollment = TRUE`, `grade_level_int >= 1`, snapshot as of previous midnight).  
3. Implement the metric in dbt, not as an ad‑hoc query.