# Troubleshooting Redis Helm Charts CrashLoop in AKS

## Overview
We encountered a `CrashLoopBackOff` issue with our Redis pods deployed using Helm charts in an Azure Kubernetes Service (AKS) cluster. This document details the steps taken to diagnose and resolve the issue.

## Environment
- **Platform**: Azure Kubernetes Service (AKS)
- **Deployment Tool**: Helm charts for Redis
- **Pod Name**: `redis-liveview-node-0`
- **Namespace**: `dummy`

## Issue Description
The Redis pod `redis-liveview-node-0` entered a `CrashLoopBackOff` state, indicating it was repeatedly crashing and restarting. This behavior suggested a configuration or data corruption issue within the Redis instance.

## Troubleshooting Steps

### 1. Accessing the Pod
We used `kubectl` to access the pod and investigate the issue:
```bash
kubectl exec -tty -i redis-client --namespace dummy -- /bin/bash
```
However, the `redis-client` pod was not found, so we targeted the `redis-liveview-node-0` pod directly:
```bash
kubectl exec -it redis-liveview-node-0 --namespace dummy -- /bin/bash
```

### 2. Checking the Filesystem
After accessing the pod, we navigated the filesystem to inspect the data and configuration:
- Changed directory to the root:
  ```bash
  cd /
  ```
- Listed directories to verify the structure:
  ```bash
  ls
  ```
  Output included standard directories like `bin`, `data`, `etc`, `home`, `lib`, `lib64`, `media`, `mnt`, `opt`, `proc`, `root`, `run`, `sbin`, `srv`, `sys`, `tmp`, `usr`, `var`.

- Navigated to the `data` directory:
  ```bash
  cd data
  ls
  ```
  Output showed a directory `appendonlydir`.

- Entered the `appendonlydir`:
  ```bash
  cd appendonlydir/
  ls
  ```
  Output listed several `.aof` files, including:
  - `appendonly.aof.688.base.rdb`
  - `appendonly.aof.689.incr.aof`
  - `appendonly.aof.611.incr.aof`
  - `appendonly.aof.612.incr.aof`
  - `appendonly.aof.613.incr.aof`
  - `appendonly.aof.614.incr.aof`
  - `appendonly.aof.609.incr.aof`
  - `appendonly.aof.609.incr.aof-backup`
  - `appendonly.aof.manifest`

### 3. Identifying the Issue
We suspected an issue with the Append-Only File (AOF) used by Redis for persistence. Specifically, the file `appendonly.aof.609.incr.aof` appeared to have a format error. To confirm, we ran the `redis-check-aof` utility to analyze and fix the file:
```bash
redis-check-aof --fix appendonly.aof.609.incr.aof
```
The tool reported:
- AOF format error in `appendonly.aof.609.incr.aof`.
- File size: 11,515,8427 bytes.
- Analysis showed the file could be truncated from 11,515,8427 bytes to 1,134,51313 bytes, removing 1,797,114 bytes of corrupted data.
- Prompted for confirmation to truncate:
  ```
  This will shrink the AOF appendonly.aof.609.incr.aof from 115158427 bytes, with 1797114 bytes, to 113451313 bytes
  continue? [y/N]: y
  ```
- After confirming with `y`, the AOF file was successfully truncated.

### 4. Verifying the Fix
After truncating the file, we exited the pod:
```bash
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

## Resolution
The `CrashLoopBackOff` issue was caused by a corrupted Append-Only File (`appendonly.aof.609.incr.aof`) in the Redis pod. Using the `redis-check-aof` utility, we identified and fixed the format error by truncating the corrupted data. After applying the fix, the pod stabilized and resumed normal operation.

## Recommendations
- Regularly monitor Redis AOF files for corruption using `redis-check-aof`.
- Consider enabling Redis persistence backup strategies to prevent data loss.
- Review Helm chart configurations for Redis to ensure optimal settings for AOF and RDB persistence.
- Set up alerts for pod crash loops in AKS to detect issues early.

## Notes
- Ensure backups of AOF files are maintained before performing any fixes.
- Test the fix in a non-production environment if possible before applying to production.
