# Time-skipping workflow test server (Temporal Java test server)

Verified against the Temporal Java SDK test server source and docs.

- Time is global to one test environment instance. Skipping happens only while no lock holds it: workflow tasks, activities, queries and similar hold a lock count.
- A workflow task takes the lock when it is scheduled and releases it when the task completes or fails. A terminate request does not release it itself, and completing a task on an already-ended workflow is rejected as not found. So never end (terminate or cancel) a workflow while one of its tasks is in flight: later sleeps can then run in real time until the test times out.
- Wait until the workflow is idle (its last history event is a timer start) before ending it. This idle check is the owner's practice, not stated in the docs.
- Await every run a test started before closing the environment.
- `Thread.sleep` is not controlled by time skipping; use the workflow's own sleep inside workflows.
- For different time behaviour across tests, run them in series or use separate environments.
