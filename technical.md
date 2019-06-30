# Technical design docs

## Current state of things about a single scan's workflow goes that way;

### Using JSONRPC for communication

* Web or CLI Client uploads a file with it's data and name (specific file names can cause differences in detection with some Anti-virus software) and a selection of profiles to scan with
* Web API creates a task for each profile and then subscribes to updates for each of these tasks using PubSub
* Coordinator queues created tasks to managers
* Managers send a request to the coordinator with task state changes
* Coordinator notifies Web API with state changes
* Coordinator re-queues any task that errors or times out according to the manager specific value
* Web API notifies Web or CLI Client of state changes through websocket

### Using PostgreSQL for communication

**Note**: Suddenly, most parties (not CLI or Web Clients) in the system get some access to the database and the coordinator is eliminated, the database acts as a coordinator

* Web or CLI Client uploads a file with it's data and name (specific file names can cause differences in detection with some Anti-virus software) and a selection of profiles to scan with
* Web API creates a task for each profile and then subscribes to updates for each of these tasks using [LISTEN](https://www.postgresql.org/docs/11/sql-listen.html)
* Managers exclusively retrieve tasks and update task row with state changes then use [NOTIFY](https://www.postgresql.org/docs/11/sql-notify.html), release task lock if times out or fails, or postgres will automatically release lock if manager disconnects due to other errors
* Web API notifies Web or CLI Client of state changes through websocket

#### Task Distribution System with PostgreSQL

```
BEGIN;
SELECT * FROM tasks FOR UPDATE SKIP LOCKED LIMIT 1;
// *process task*
UPDATE tasks SET result = $TASK_RESULT WHERE id = $TASK_ID;
COMMIT;
```