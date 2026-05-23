# Questions and Answers

#### How would you test that every student in *dim_students* exists in the source enrollment data? Why is this test important?
- The test is called referential integrity or relationship test
- **How the test works?**
  - Take all *student_id* values from the *dim_students*
  - Check if each one exists in the *stg_sis_enrollments* table
  - If any *student_id* from the dimension is missing from the staging, the test fails
- **Why this is important:**
  1. Prevents phantom students
  2. Maintains trust downstream
  3. Catches data pipeline bugs early
  4. Documentation expctations

#### The superintendent asks, "How many students are currenlty enrolled across all Texas campuses?" What questions do you ask before writing any SQL?
1. Time definition - "currently"
2. Enrollment definition - what does "enrolled" mean?
3. Campus scope - "Texas campuses"
4. Student uniqueness - how to count?
5. Exclusion - any grade or programs to exclude?

#### What is the difference between a view, a table, and an incremental materialization in dbt? Which would you use for each of your three models and why?
- View: A stored SQL query without the source data
- Table: A physical copy of the data stored in the warehouse
- Incremental: A hybrid where dbt builds a table by inserting new rows and updating changed rows based on a unique key.
- Choices for the three models:
  - Staging model
  - Dimension model
  - Fact model
 
#### How do you handle late-arriving data - for example, an exit date that comes in a week after the student actually withdrew?
- Stragies to handle this:
  1. Incremental fact model with a lookback window
  2. Handle both entry and exit dates
  3. For the dimension table
  4. Periodic full refresh
  5. Document the latency
 
#### What is a surrogate key and why would you add one to the staging model? What natural composite key would you use if you dind't have a surrogate?
- Why add a surrogate key to the staging model:
  1. The raw CSV has no unique row identifier
  2. Surrogate keys are stable
  3. Performance
  4. Auditability
- Natural composite key if no surrogate

#### The custom test for overlapping enrollments passes, but a principal reports a student still appears at two schools on the same day in their dashboard. What could have gone wrong?
1. The dashboard is reading from the wrong table
2. The test only checks for overlaps at different schools, but the student has two overlapping rows at the same school
3. The overlap spans across a date boundary where the test's inequality logic fails
4. The dashboard's date filter is using a different time zone
5. The fact table's overlap resolution logic was updated, bu the dashboard was not refreshed
