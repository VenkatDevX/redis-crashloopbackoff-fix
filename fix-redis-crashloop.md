**Title: Comprehensive Guide to Troubleshooting and Fixing Redis CrashLoopBackOff in AKS Using Helm Chart**

### Issue:
The Redis pods deployed using Helm charts in AKS went into `CrashLoopBackOff`. Investigation was conducted to check and fix the issue.

### Steps Taken:

#### 1. Accessing the Redis Pod
- Tried to access a Redis client pod, but it was not found:
  ```sh
  kubectl exec --tty -i redis-client \ --namespace dummy -- bash
  ```
  **Error:** `pods "redis-client" not found`

- Attempted to access the Redis liveview node and encountered an issue due to incorrect pod selection:
  ```sh
  kubectl exec -it redis-liveview-node-0 -n dummy -c sentinel -- /bin/bash
  ```
  **Error:** `container sentinel is not valid for pod redis-liveview-node-0`

- Corrected the command and successfully accessed the pod:
  ```sh
  kubectl exec -it redis-liveview-node-0 -n dummy -c sentinel -- /bin/bash
  ```

#### 2. Navigating to Redis Data Directory
- Listed directories inside the container:
  ```sh
  ls
  ```
- Moved into the Redis data directory:
  ```sh
  cd /data
  ls
  ```
  **Output:** Found `appendonlydir`, `dump.rdb`, and `lost+found`.

#### 3. Checking and Fixing Append-Only File (AOF)
- Navigated to `appendonlydir`:
  ```sh
  cd appendonlydir/
  ls
  ```
  **Output:** Found multiple `appendonly.aof` files.

- Created a backup of the affected AOF file before fixing:
  ```sh
  cp appendonly.aof.609.incr.aof appendonly.aof.609.incr.aof-backup
  ```

- Attempted to run `redis-check-aof` with `sudo`, but encountered an error (`sudo: command not found`).
  Resolved by running directly:
  ```sh
  redis-check-aof --fix appendonly.aof.609.incr.aof
  ```
  **Output:**
  - Found format error in `appendonly.aof.609.incr.aof`
  - Shrunk AOF file from `115158427 bytes` to `113451313 bytes`
  - Successfully truncated the AOF file

#### 4. Exiting and Verifying Redis Pod Status
- Exited the Redis pod:
  ```sh
  exit
  ```
- Checked the status of Redis pods:
  ```sh
  kubectl get pods -n dummy
  ```
- If pods were still in `CrashLoopBackOff`, restarted them:
  ```sh
  kubectl delete pod redis-liveview-node-0 -n dummy
  ```
  Alternatively, restarted the entire deployment:
  ```sh
  kubectl rollout restart statefulset redis -n dummy
  ```

### Outcome:
The corrupted append-only file (`AOF`) was successfully fixed, and Redis pods were able to restart without issues. Monitoring was continued to ensure stability.
