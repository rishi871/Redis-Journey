---
name: troubleshooting-stuck-state-machines
description: Diagnostic workflow for Redis Enterprise databases stuck in state machine errors (active-change-pending, pending, etc.)
---

# Stuck State Machine Diagnostic Workflow

## Quick Start

When a customer reports a database stuck in `active-change-pending`, `pending`, or similar status:

1. **Get the support package**: Ask customer to generate debuginfo (`rladmin cluster debug_info`)
2. **Attach to conversation**: Use `@/path/to/debuginfo.XXXXX` syntax
3. **Let Augment AI analyze**: I will systematically check CCS data and logs
4. **Follow resolution**: Based on the specific failure pattern identified

## What Augment AI Will Check (Automatically)

When you attach a support package with `@`, I will:

- [ ] Identify affected databases from CCS data
- [ ] Extract state machine status (`cnm_exec_sm`, `cnm_exec_state`, error attributes)
- [ ] Analyze `cnm_exec.log` for state transitions and failures
- [ ] Check component-specific logs (redis, dmcproxy, syncer, etc.)
- [ ] Look for resource issues (port conflicts, dead nodes, provisioning errors)
- [ ] Cross-reference with known state machine patterns
- [ ] Provide root cause analysis and resolution steps

---

## Support Package Structure

```
debuginfo.XXXXX/
├── database_X/
│   ├── database_X_ccs_info.txt          ← **START HERE** for state machine status
│   ├── database_X.rladmin
│   └── database_X.info
├── node_Y/
│   ├── logs/
│   │   ├── cnm_exec.log                 ← State machine transitions
│   │   ├── redis_mgr.log                ← Redis process management
│   │   ├── redis-<shard-id>.log         ← Shard-specific errors
│   │   ├── db_controller.log            ← Database controller operations
│   │   ├── shard_mgr.log                ← Shard placement/resharding
│   │   ├── dmcproxy.log                 ← Proxy configuration
│   │   └── syncer-<bdb-uid>.log         ← Replication issues
│   ├── node_Y_sys_info.txt              ← Process list, port bindings, resources
│   ├── node_Y.ccs                       ← Full CCS dump
│   └── redis_X.txt                      ← Redis INFO output
```

---

## Critical CCS Attributes for State Machine Diagnosis

### Primary Indicators

| Attribute | Meaning | Example Values |
|-----------|---------|----------------|
| `status` | Database status | `active`, `active-change-pending`, `pending`, `creation-failed` |
| `cnm_exec_sm` | Current state machine name | `SMUpdateBDB`, `SMCreateBDB`, `SMRedisFailover` |
| `cnm_exec_state` | Current state within SM | `error`, `update_redis`, `provision`, etc. |
| `cnm_exec_sm_stack` | Nested state machines | Shows parent/child SM relationships |
| `action_uid` | Unique operation identifier | UUID for tracking in logs |

### Error Indicators

| Attribute | Meaning | When Present |
|-----------|---------|--------------|
| `__sm_error` | Error message/reason | Always present when SM fails |
| `__sm_failed_state` | The state that failed | Shows where SM got stuck |
| `__sm_retry_state` | State to retry from | For retryable errors only |
| `stuck_sm__` | Previous stuck SM name | If SM was stuck before |

### Example CCS Output

```
status: active-change-pending
cnm_exec_sm: SMUpdateBDB:updates=reshard,bind_endpoint,sharding,proxy_policy:params=reshard=params,new_shards=1
cnm_exec_state: error
__sm_failed_state: update_redis
__sm_retry_state: update_dns
__sm_error: redis:4: cnm_exec:1 remote action failed in SMUpdateBDB [update_redis] , setting retryable error state.
action_uid: c556ecd8-4db7-4d87-a3cf-93e6a3b3827f
```

**Interpretation**: Database stuck in SMUpdateBDB, failed at `update_redis` state, retryable from `update_dns`

---

## Common Stuck State Machine Patterns

### Pattern 1: Port Conflict During Resharding/Update

**Signature:**
- `__sm_failed_state: update_redis`
- cnm_exec.log: `pidfile error: [Errno 2] No such file or directory: '/var/run/redis/redis-X.pid'`
- cnm_exec.log: `redis:X is not running after all retries`
- redis-X.log: `bind: Address already in use` or `Failed listening on port XXXXX (tcp), aborting`
- sys_info.txt: Shows redis process already listening on the port

**Root Cause:** Old redis process still running and blocking the port, preventing new instance from starting

**Resolution:**
1. Identify the stale process: `ps aux | grep redis-X` or check sys_info.txt
2. Kill the process: `sudo kill -TERM <PID>` or `sudo supervisorctl stop redis-X`
3. Clean PID file: `sudo rm -f /var/run/redis/redis-X.pid`
4. Retry operation: `rladmin retry bdb db:X`

**Code Reference:** `cnm/cnm/statemachine/update.py` - `update_redis` state

---

### Pattern 2: Provisioning Failure (No Available Resources)

**Signature:**
- `__sm_failed_state: provision` or `shards_placement`
- `__sm_error: no available nodes in cluster!` or `no available redis ports in cluster!` or `no available proxies for binding an endpoint`
- cnm_exec.log: `ProvisioningError`

**Root Cause:** Insufficient cluster resources (nodes, ports, or proxies) for the requested operation

**Resolution:**
1. **No available nodes**: Add nodes to cluster (`rladmin cluster join nodes <ip>`)
2. **No available ports**: Free up ports or add nodes
3. **No available proxies**: Increase proxy count or add nodes
4. Retry operation after adding resources

**Code Reference:** `cnm/cnm/statemachine/helpers.py` - `ProvisioningHelper`, `ProvisioningError`

---

### Pattern 3: Dead Node During Failover

**Signature:**
- `cnm_exec_sm: SMRedisFailover`
- `__sm_failed_state: promote_slave`, `drain_master`, or `demote_live_master`
- cnm_exec.log: `remote action failed` with node timeout or connection errors
- cluster_wd.log or node_wd.log: Node marked as dead or unreachable

**Root Cause:** Target node is unreachable, dead, or services are down

**Resolution:**
1. Check node health: `rladmin status nodes`
2. If node is dead: Remove from cluster or recover it
3. If services down: Restart CNM services (`supervisorctl restart all`)
4. Retry failover or let watchdog complete it automatically

**Code Reference:** `cnm/cnm/statemachine/failover.py` - SMRedisFailover states

---

### Pattern 4: Configuration Validation Failure

**Signature:**
- `__sm_failed_state: prepare` or `init`
- `__sm_error` contains validation error (e.g., "invalid memory size", "invalid parameter")
- cnm_exec.log: State machine fails immediately at start, before any resource allocation

**Root Cause:** Invalid configuration parameters provided to the operation

**Resolution:**
1. Review the error message for specific validation failure
2. Fix configuration via API or rladmin
3. Retry operation with corrected parameters

**Code Reference:** `cnm/cnm/statemachine/base.py` - validation logic in `prepare`/`init` states

---

### Pattern 5: Replication Setup Failure

**Signature:**
- `__sm_failed_state: replication` or `hidden_replication`
- `__sm_error` mentions syncer errors or replication configuration issues
- syncer-X.log: Connection errors, authentication failures, or sync errors

**Root Cause:** Cannot establish replication (replica-of or CRDB) due to connectivity, authentication, or configuration issues

**Resolution:**
1. Check syncer logs for specific error
2. Verify network connectivity between clusters
3. Check authentication credentials
4. Verify source database is accessible
5. Retry operation after fixing connectivity/auth issues

**Code Reference:** `cnm/cnm/statemachine/update.py` - `replication` and `hidden_replication` states

---

### Pattern 6: Endpoint Binding Failure

**Signature:**
- `__sm_failed_state: bind_endpoint`
- `__sm_error` mentions proxy or endpoint errors
- dmcproxy.log: Proxy configuration errors or binding failures

**Root Cause:** Cannot bind endpoint to proxies (proxy unavailable, port conflict, or configuration error)

**Resolution:**
1. Check proxy availability: `rladmin status proxies`
2. Verify proxy policy is valid
3. Check for port conflicts on proxy nodes
4. Restart dmcproxy if needed: `supervisorctl restart dmcproxy`
5. Retry operation

**Code Reference:** `cnm/cnm/statemachine/update.py` - `bind_endpoint` state

---

## Step-by-Step Diagnostic Workflow

### Step 1: Identify Affected Database(s)

**Files to check:**
- `database_X/database_X_ccs_info.txt` for each database directory

**What to look for:**
```bash
grep -E "status:|cnm_exec_state:|__sm_error" database_*/database_*_ccs_info.txt
```

**Expected output:**
```
database_2/database_2_ccs_info.txt:status: active-change-pending
database_2/database_2_ccs_info.txt:cnm_exec_state: error
database_2/database_2_ccs_info.txt:__sm_error: redis:4: cnm_exec:1 remote action failed
```

**Action:** Note the database UID (e.g., `bdb:2`) and proceed to Step 2

---

### Step 2: Extract State Machine Details

**From the CCS file, extract and note:**

- [ ] `cnm_exec_sm` - Which state machine is running? (e.g., SMUpdateBDB, SMCreateBDB)
- [ ] `cnm_exec_state` - Current state (e.g., `error`, or specific state name)
- [ ] `__sm_failed_state` - Where did it fail? (e.g., `update_redis`, `provision`)
- [ ] `__sm_retry_state` - Can it be retried? (present = retryable)
- [ ] `__sm_error` - Full error message (contains root cause clues)
- [ ] `action_uid` - Unique identifier for tracking in logs
- [ ] `redis_list` - Which redis shards are involved? (e.g., `4` or `4,5,6`)
- [ ] `endpoint_list` - Which endpoints? (e.g., `2:1`)

**Example extraction:**
```
State Machine: SMUpdateBDB (resharding operation)
Current State: error
Failed State: update_redis
Retry State: update_dns (retryable!)
Error: redis:4: cnm_exec:1 remote action failed
Action UID: c556ecd8-4db7-4d87-a3cf-93e6a3b3827f
Redis Shards: 4
```

---

### Step 3: Analyze State Machine Transitions in Logs

**File:** `node_X/logs/cnm_exec.log` (check master node first, usually node:1)

**Search patterns:**

```bash
# Find state machine start (use action_uid from CCS)
grep "c556ecd8-4db7-4d87-a3cf-93e6a3b3827f" cnm_exec.log

# Or search by BDB UID
grep "bdb:2" cnm_exec.log | grep -E "STATEMACHINE|entering state|fulfilled|failed"

# Find specific state machine start
grep "=== \[bdb:2\] STATEMACHINE \[SMUpdateBDB\] STARTED ===" cnm_exec.log

# Find state transitions
grep "SMUpdateBDB: bdb:2\[.*\]: entering state" cnm_exec.log

# Find failures
grep "SMUpdateBDB: bdb:2\[.*\]: state failed" cnm_exec.log
grep "remote action failed" cnm_exec.log | grep "bdb:2"

# Find error state
grep "Error found when handling state machine" cnm_exec.log | grep "bdb:2"
```

**Timeline reconstruction:**

1. **When did SM start?** Look for `STATEMACHINE [SMName] STARTED` with matching action_uid
2. **Which states succeeded?** Look for `fulfilled, calling next state`
3. **Which state failed?** Look for `state failed` or `remote action failed`
4. **Were there retries?** Look for multiple attempts at the same state
5. **What was the last state entered?** Should match `__sm_failed_state` from CCS

**Example timeline:**
```
16:32:55 - SMUpdateBDB STARTED (action_uid: c556ecd8...)
16:32:55 - [prepare] entering state → fulfilled
16:32:55 - [flush] entering state → fulfilled
16:32:55 - [update_dns] entering state → fulfilled
16:32:55 - [update_redis] entering state
16:32:55 - redis:4: notifying cnm_exec:1, op=update_config
16:32:55 - redis:4: pidfile error: [Errno 2] No such file or directory
16:33:06 - redis:4 is not running after all retries
16:33:06 - ERROR: remote action failed in SMUpdateBDB [update_redis]
```

---

### Step 4: Check Component-Specific Logs

**Based on the failed state, check the appropriate logs:**

| Failed State | Log to Check | What to Look For |
|--------------|--------------|------------------|
| `provision`, `shards_placement` | `shard_mgr.log`, `resource_mgr.log` | Resource allocation errors, "no available nodes/ports" |
| `update_redis`, `restart` | `redis-X.log`, `redis_mgr.log` | Redis startup failures, port conflicts, config errors |
| `update_dmc`, `bind_endpoint` | `dmcproxy.log` | Proxy configuration errors, binding failures |
| `replication`, `hidden_replication` | `syncer-<bdb-uid>.log` | Replication setup failures, connectivity issues |
| `promote_slave`, `drain_master` | `cluster_wd.log`, `node_wd.log` | Node health, watchdog actions, failover events |
| `config_redis_acl` | `redis-X.log` | ACL configuration errors |
| `reshard`, `adjust_to_shards_blueprint` | `shard_mgr.log`, `redis_mgr.log` | Resharding errors, slot migration issues |

**Example: Checking redis-4.log for update_redis failure:**
```bash
grep -E "Fatal|error|Error|failed|Failed|bind|Address already in use" redis-4.log
```

**Look for:**
- Startup errors (bind failures, config errors)
- Crash messages
- Port conflicts
- Permission issues

---

### Step 5: Check System Resources and Process State

**File:** `node_X/node_X_sys_info.txt`

**Check for:**

1. **Port conflicts:**
   ```bash
   grep ":<port>" node_X_sys_info.txt
   # Example: grep ":29051" node_1_sys_info.txt
   ```
   Look for multiple processes listening on the same port

2. **Stale processes:**
   ```bash
   grep "redis-X" node_X_sys_info.txt
   ```
   Check if redis process is running when it shouldn't be (or vice versa)

3. **Disk space:**
   ```bash
   grep -A 10 "Filesystem" node_X_sys_info.txt
   ```
   Look for filesystems at 100% usage

4. **Memory:**
   ```bash
   grep -A 5 "MemTotal\|MemFree" node_X_sys_info.txt
   ```
   Check for OOM conditions

5. **PID file location:**
   ```bash
   grep "pidfile" node_X/redis_X.txt
   ```
   Verify PID file path matches what cnm_exec expects

---

### Step 6: Determine Resolution Strategy

**Decision tree:**

```
Is __sm_retry_state present in CCS?
├─ YES → Error is RETRYABLE
│   ├─ Is root cause identified and fixable?
│   │   ├─ YES → Fix root cause, then retry
│   │   │   └─ Examples: Kill stale process, free disk space, add nodes
│   │   └─ NO → Need more investigation (go back to Step 3-5)
│   │
│   └─ Has retry been attempted multiple times already?
│       ├─ NO → Safe to retry: `rladmin retry bdb db:X`
│       └─ YES → Consider smtool.py or escalation
│
└─ NO → Error is NOT retryable
    ├─ Check if smtool.py has recovery action
    │   └─ Examples: ClearRedisErrorsAction, DeleteStuckBDBAction
    └─ If no recovery action available → ESCALATE
```

**Key questions to answer:**
1. What is the root cause? (port conflict, resource shortage, dead node, etc.)
2. Can it be fixed without data loss? (preferred)
3. Is the error retryable? (check `__sm_retry_state`)
4. Has it been retried already? (check logs for previous retry attempts)
5. Is customer data at risk? (if yes, escalate immediately)

---

## Resolution Actions

### Action 1: Retry State Machine (For Retryable Errors)

**When to use:** `__sm_retry_state` is present in CCS

**Prerequisites:**
- Root cause has been identified and fixed
- No ongoing operations on the database
- Cluster is healthy

**Via rladmin:**
```bash
rladmin retry bdb db:X
```

**Via API:**
```bash
curl -k -u "user@example.com:password" -X POST \
  https://<cluster-fqdn>:9443/v1/bdbs/X/actions/retry
```

**Monitor progress:**
```bash
# Watch state machine transitions
tail -f /var/opt/redislabs/log/cnm_exec.log | grep "bdb:X"

# Check database status
rladmin status databases | grep "db:X"

# Or via API
curl -k -u "user@example.com:password" \
  https://<cluster-fqdn>:9443/v1/bdbs/X | jq '.status'
```

**Expected behavior:**
- State machine resumes from `__sm_retry_state`
- If successful: `status` changes from `active-change-pending` to `active`
- If failed again: New error appears in `__sm_error`

---

### Action 2: Manual Intervention (Fix Root Cause First)

#### Scenario A: Port Conflict

**Symptoms:**
- `pidfile error` in cnm_exec.log
- `Address already in use` in redis-X.log
- Process already listening on port in sys_info.txt

**Resolution:**
```bash
# 1. Find the process using the port
sudo lsof -i :PORT
# Or from support package: grep ":PORT" node_X_sys_info.txt

# 2. Kill the stale process
sudo kill -TERM <PID>
# Or use supervisorctl
sudo supervisorctl stop redis-X

# 3. Clean up PID file
sudo rm -f /var/run/redis/redis-X.pid

# 4. Verify port is free
sudo lsof -i :PORT  # Should return nothing

# 5. Retry the operation
rladmin retry bdb db:X
```

---

#### Scenario B: Resource Shortage (No Available Nodes/Ports)

**Symptoms:**
- `no available nodes in cluster!` in `__sm_error`
- `no available redis ports in cluster!`
- `ProvisioningError` in cnm_exec.log

**Resolution:**

**Option 1: Add nodes to cluster**
```bash
# Add new node
rladmin cluster join nodes <new-node-ip> username <user> password <pass>

# Verify node is added
rladmin status nodes

# Retry operation
rladmin retry bdb db:X
```

**Option 2: Free up resources**
```bash
# Delete unused databases
rladmin delete db <unused-db-name>

# Or reduce shard count on other databases
rladmin tune db <other-db> shards_count <lower-number>

# Retry operation
rladmin retry bdb db:X
```

**Option 3: Reduce requested resources**
```bash
# If creating/updating database, reduce shard count
rladmin tune db db:X shards_count <lower-number>
```

---

#### Scenario C: Dead Node During Failover

**Symptoms:**
- `remote action failed` with node timeout
- Node marked as dead in cluster_wd.log
- Cannot reach node

**Resolution:**

**Option 1: Recover the node**
```bash
# SSH to the dead node
ssh <node-ip>

# Check service status
sudo supervisorctl status

# Restart all services
sudo supervisorctl restart all

# Verify node is back
rladmin status nodes
```

**Option 2: Remove dead node (if unrecoverable)**
```bash
# Remove node from cluster
rladmin node <node-id> remove

# Cluster will automatically rebalance
# Monitor with: rladmin status
```

**Option 3: Let watchdog complete failover**
- If node is permanently dead, watchdog will eventually complete failover
- Monitor cluster_wd.log for automatic recovery
- May take several minutes

---

#### Scenario D: Stale Redis Process

**Symptoms:**
- Redis process running but not responding
- PID file missing or stale
- State machine cannot communicate with redis

**Resolution:**
```bash
# 1. Stop the redis process
sudo supervisorctl stop redis-X

# 2. Verify process is stopped
ps aux | grep redis-X  # Should return nothing

# 3. Clean up stale files
sudo rm -f /var/run/redis/redis-X.pid
sudo rm -f /var/opt/redislabs/run/redis-X.sock

# 4. Start redis (if needed)
sudo supervisorctl start redis-X

# 5. Retry operation
rladmin retry bdb db:X
```

---

### Action 3: Use smtool.py (Advanced Recovery)

**⚠️ WARNING:** smtool.py actions can cause **DATA LOSS**. Always:
1. Backup data if possible
2. Get customer approval
3. Document the action taken
4. Use as last resort before escalation

**Location:** `/opt/redislabs/bin/smtool.py` (on cluster nodes)

**Common actions:**

#### List available actions:
```bash
python3 /opt/redislabs/bin/smtool.py --list-actions
```

#### Clear redis errors and retry:
```bash
# Use when redis shards have errors preventing state machine progress
python3 /opt/redislabs/bin/smtool.py \
  --action ClearRedisErrorsAction \
  --bdb-uid X
```

#### Delete stuck BDB (DESTRUCTIVE):
```bash
# ⚠️ DATA LOSS! Only use when database is unrecoverable
python3 /opt/redislabs/bin/smtool.py \
  --action DeleteStuckBDBAction \
  --bdb-uid X
```

#### Force state machine completion:
```bash
# Use when state machine is stuck in non-error state
python3 /opt/redislabs/bin/smtool.py \
  --action ForceCompleteAction \
  --bdb-uid X
```

**When to use smtool.py:**
- Multiple retry attempts have failed
- State machine is in an inconsistent state
- Manual intervention hasn't resolved the issue
- Before escalating to Engineering (as a last attempt)

**Code Reference:** `cnm/utils/smtool.py` - Contains all available recovery actions

---

### Action 4: Restart CNM Services (If State Machine is Unresponsive)

**When to use:**
- State machine is stuck and not responding to retries
- cnm_exec service appears hung
- After fixing root cause but retry still fails

**⚠️ WARNING:** Restarting CNM services may interrupt ongoing operations on other databases

**Procedure:**
```bash
# On the master node (usually node:1)
sudo supervisorctl restart cnm_exec
sudo supervisorctl restart redis_mgr

# Wait 30 seconds for services to stabilize
sleep 30

# Check service status
sudo supervisorctl status | grep cnm

# Retry the operation
rladmin retry bdb db:X
```

**Monitor:**
```bash
# Watch cnm_exec restart
tail -f /var/opt/redislabs/log/cnm_exec.log

# Look for: "cnm_exec starting" and "Scan finished, entering main loop"
```

---

### Action 5: Escalation to Engineering

**Escalate when:**
- Multiple retry attempts have failed
- smtool.py actions don't resolve the issue
- Error pattern is unknown/undocumented
- Customer data is at risk
- State machine has been stuck for >24 hours
- Root cause cannot be identified

**Information to provide in escalation:**

1. **Database details:**
   - BDB UID: `bdb:X`
   - Database name: `<name>`
   - Database type: redis/memcached/CRDB

2. **State machine details:**
   - State machine name: `<cnm_exec_sm value>`
   - Failed state: `<__sm_failed_state value>`
   - Error message: `<full __sm_error value>`
   - Action UID: `<action_uid value>`

3. **Timeline:**
   - When did the operation start?
   - When did it fail?
   - How many retry attempts?
   - What manual interventions were attempted?

4. **Support package:**
   - Path to support package
   - Timestamp of support package generation
   - Highlight relevant log excerpts

5. **Impact:**
   - Is database accessible to clients?
   - Is data at risk?
   - Are other databases affected?
   - Customer severity level

**Escalation template:**
```
Subject: Stuck State Machine - bdb:X in <state_machine_name>

Database: bdb:X (<database_name>)
State Machine: <cnm_exec_sm>
Failed State: <__sm_failed_state>
Error: <__sm_error>

Timeline:
- Operation started: <timestamp>
- Failed at: <timestamp>
- Retry attempts: <count>

Actions attempted:
1. <action 1>
2. <action 2>
...

Support package: <path or attachment>
Relevant logs: See cnm_exec.log lines <start>-<end>

Impact: <describe customer impact>
```

---

## State Machine Code References

### State Machine Implementations

#### Base Classes
- **File:** `cnm/cnm/statemachine/base.py`
- **Classes:** `BaseStateMachine`, `StateMachine`
- **Key methods:**
  - `set_error_state()` - Sets state machine to error state
  - `has_error()` - Checks if state machine has error
  - `_perform_next_state()` - Transitions to next state
  - `handle_state_machine()` - Main state machine handler

#### SMCreateBDB (Database Creation)
- **File:** `cnm/cnm/statemachine/create.py`
- **States:** `init`, `provision`, `create_redis`, `bind_endpoint`, `update_dmc`, `config_redis_acl`, `created`
- **Common failures:**
  - `provision` - No available resources
  - `create_redis` - Redis startup failures
  - `bind_endpoint` - Proxy binding failures

#### SMUpdateBDB (Database Update/Resharding)
- **File:** `cnm/cnm/statemachine/update.py`
- **States:** `prepare`, `restart`, `flush`, `pause_persistence`, `replication`, `hidden_replication`, `update_security`, `config_encryption`, `update_dns`, `update_redis`, `shards_placement`, `proxy_policy`, `update_endpoints_ip_type`, `reshard`, `adjust_to_shards_blueprint`, `bind_endpoint`, `update_dmc`, `config_redis_acl`, `resume_persistence`, `updated`
- **Common failures:**
  - `update_redis` - Redis configuration update failures, port conflicts
  - `reshard` - Resharding/slot migration failures
  - `shards_placement` - Cannot place shards on nodes
  - `bind_endpoint` - Endpoint binding failures

#### SMRedisFailover (Failover Operations)
- **File:** `cnm/cnm/statemachine/failover.py`
- **States:** `prepare`, `check_slave`, `disable_fullsync`, `stop_forwarding`, `stop_active_expire`, `stop_wd`, `drain_master`, `promote_slave`, `enable_fullsync`, `demote_live_master`, `kill_failed_master`, `notify_dmc`, `rebind_endpoints`, `resume_active_expire`, `start_failed_master`, `start_wd`, `done`
- **Common failures:**
  - `promote_slave` - Slave promotion failures
  - `drain_master` - Cannot drain master
  - `demote_live_master` - Master demotion failures

#### SMDeleteBDB (Database Deletion)
- **File:** `cnm/cnm/statemachine/delete.py`
- **States:** `init`, `undo_stuck_sm`, `stop_syncer`, `unbind_endpoints`, `delete_redis`, `delete_bdb`, `deleted`
- **Special state:** `undo_stuck_sm` - Handles cleanup of stuck state machines

#### SMRecoverBDB (Database Recovery)
- **File:** `cnm/cnm/statemachine/recover.py`
- **States:** Recovery-specific states for database restoration
- **Common failures:** Resource allocation, data restoration errors

#### SMUpgradeBDB (Database Upgrade)
- **File:** `cnm/cnm/statemachine/upgrade.py`
- **States:** Similar to SMUpdateBDB with version upgrade logic
- **Common failures:** Version compatibility issues, restart failures

---

### Helper Modules

#### Provisioning Helper
- **File:** `cnm/cnm/statemachine/helpers.py`
- **Classes:** `ProvisioningHelper`, `ProvisioningError`
- **Functionality:** Resource allocation logic (nodes, ports, proxies)
- **Error messages:**
  - `"no available nodes in cluster!"`
  - `"no available redis ports in cluster!"`
  - `"no available proxies for binding an endpoint"`

#### BDB Object
- **File:** `cnm/cnm/ccs/objects/bdb.py`
- **Key methods:**
  - `has_statemachine_error()` - Checks if BDB has SM error
  - `get_failed_state()` - Returns `__sm_failed_state`
  - `retryable_error_state()` - Checks if error is retryable
  - `initiate_retry()` - Initiates retry operation
- **Schema:** `cnm/ccs_schema/bdb.json` - Defines all CCS attributes

#### Recovery Tool
- **File:** `cnm/utils/smtool.py`
- **Actions:**
  - `ClearRedisErrorsAction` - Clears redis error states
  - `DeleteStuckBDBAction` - Deletes stuck database (DESTRUCTIVE)
  - `ForceCompleteAction` - Forces state machine completion
  - `RetryStatemachineAction` - Retries state machine
  - And more... (use `--list-actions` to see all)

---

### Log Patterns Reference

#### State Machine Lifecycle
```
# State machine start
=== [bdb:X] STATEMACHINE [SMName] STARTED ===
=== [bdb:X] Action uid: <uuid> ===
=== [bdb:X] Parameters: <json-params> ===

# State entry
SMName: bdb:X[state_name]: entering state

# State completion
SMName: bdb:X[state_name]: state done.
SMName: bdb:X[state_name]: fulfilled, calling next state.

# State failure
SMName: bdb:X[state_name]: state failed

# Nested state machine
Invoking nested call on bdb:X => [NestedSMName], Arguments [...]

# State machine end (success)
=== [bdb:X] STATEMACHINE [SMName] ENDED SUCCESSFULLY ===

# State machine end (error)
Error found when handling state machine! Returning...
```

#### Error Patterns
```
# Remote action failure
redis:X: cnm_exec:Y remote action failed in SMName [state_name]

# Retryable error
setting retryable error state.
retry from state:<state> reason:<error> failed_state:<state>

# Provisioning error
ProvisioningError: no available nodes in cluster!

# PID file error
redis:X: pidfile error: [Errno 2] No such file or directory: '/var/run/redis/redis-X.pid'

# Redis not running
redis:X is not running after all retries.

# Deferred operation failure
Deferred op:<uuid>(<OpName>) failed. Current op info: {...}
```

#### Color Patterns (from `cnm/etc/shlog/cnm_exec.conf`)
```
# State machine transitions (colored in logs)
(SM\w+): bdb:\d+\[([\w_]+)\]: (entering state)
(SM\w+): bdb:\d+\[([\w_]+)\]: (fulfilled, calling next state)
(SM\w+): bdb:\d+\[([\w_]+)\]: (state failed)
```

---

## Real-World Example: Port Conflict During Resharding

This is the actual case we diagnosed together:

### Customer Report
```
Database stuck in 'ACTIVE-CHANGE-PENDING' status after enabling cluster sharding on February 20, 2026.

db:2  NEWFDB redis active-change-pending 1      dense     disabled    disabled    redis-12000.nou.com:12000
RETRYABLE ERROR: redis:4: cnm_exec:1 remote action failed in SMUpdateBDB:updates=reshard,bind_endpoint,sharding,proxy_policy:params=reshard=params,new_shards=1,reuse_slave=no,endpoint=2-1,proxy_policy=single [update_redis] , setting retryable error state.
```

### Diagnostic Process

**Step 1: CCS Analysis**
```
# From database_2/database_2_ccs_info.txt
status: active-change-pending
cnm_exec_sm: SMUpdateBDB:updates=reshard,bind_endpoint,sharding,proxy_policy
cnm_exec_state: error
__sm_failed_state: update_redis
__sm_retry_state: update_dns
__sm_error: redis:4: cnm_exec:1 remote action failed in SMUpdateBDB [update_redis]
redis_list: 4
```

**Interpretation:** Retryable error in SMUpdateBDB at `update_redis` state, affecting redis:4

**Step 2: Log Analysis**
```
# From node_1/logs/cnm_exec.log
2026-02-20 16:32:55,835 INFO: SMUpdateBDB: bdb:2[update_redis]: entering state
2026-02-20 16:32:55,837 INFO: redis:4: notifying cnm_exec:1, op=update_config
2026-02-20 16:32:55,978 INFO: redis:4: pidfile error: [Errno 2] No such file or directory: '/var/run/redis/redis-4.pid'
...
2026-02-20 16:33:06,536 ERROR: redis:4 is not running after all retries.
2026-02-20 16:33:06,540 ERROR: redis:4: cnm_exec:1 remote action failed in SMUpdateBDB [update_redis]
```

**Step 3: Redis Log Analysis**
```
# From node_1/logs/redis-4.log
3725842:M 20 Feb 2026 16:42:50.191 # Warning: Could not create server TCP listening socket *:29051: bind: Address already in use
3725842:M 20 Feb 2026 16:42:50.191 # Failed listening on port 29051 (tcp), aborting.
```

**Step 4: System Info Check**
```
# From node_1/node_1_sys_info.txt
redisla+ 3664703  0.0  0.0 143224  9392 ?        Ssl  Feb20   3:06 /u02/nou_redis/redis_install/redislabs/bin/redis-server-7.2 /u02/nou_redis/redis_var/redislabs/redis/redis-4.conf *:29051
tcp   LISTEN    0      511                  0.0.0.0:29051                 0.0.0.0:*
```

### Root Cause
Old redis:4 process (PID 3664703) still running and holding port 29051, preventing new instance from starting during resharding operation.

### Resolution
```bash
# 1. Kill stale process
sudo kill -TERM 3664703

# 2. Clean PID file
sudo rm -f /var/run/redis/redis-4.pid

# 3. Retry operation
rladmin retry bdb db:2

# 4. Monitor
tail -f /var/opt/redislabs/log/cnm_exec.log | grep "bdb:2"
```

### Outcome
State machine resumed from `update_dns` state, successfully completed resharding, database returned to `active` status.

---

## Quick Reference Card

### CCS Attributes Cheat Sheet
```
status                  → Database status (active, active-change-pending, pending, etc.)
cnm_exec_sm            → Current state machine name
cnm_exec_state         → Current state (error = stuck)
__sm_failed_state      → Where it failed
__sm_retry_state       → Can retry? (present = yes)
__sm_error             → Error message
action_uid             → Track in logs
redis_list             → Affected shards
endpoint_list          → Affected endpoints
```

### Log Search Commands
```bash
# Find stuck databases
grep -E "status:|cnm_exec_state:|__sm_error" database_*/database_*_ccs_info.txt

# Find state machine timeline
grep "<action_uid>" cnm_exec.log

# Find failures
grep "remote action failed\|state failed\|Error found" cnm_exec.log

# Find redis errors
grep -E "Fatal|failed|Failed|bind|Address already in use" redis-X.log

# Find port conflicts
grep ":<port>" node_X_sys_info.txt
```

### Resolution Decision Tree
```
Retryable? (__sm_retry_state present)
├─ YES
│  ├─ Root cause fixed? → rladmin retry bdb db:X
│  └─ Root cause unknown? → Investigate more
└─ NO
   ├─ smtool.py action available? → Use smtool.py
   └─ No action available? → ESCALATE
```

---

## Additional Resources

- **Main troubleshooting guide:** `.augment/troubleshooting.md`
- **State machine code:** `cnm/cnm/statemachine/`
- **Recovery tool:** `cnm/utils/smtool.py`
- **CCS schema:** `cnm/ccs_schema/bdb.json`
- **Log configuration:** `cnm/etc/shlog/cnm_exec.conf`

---

**Last Updated:** February 2026
**Maintainer:** Redis Technical Support Team


