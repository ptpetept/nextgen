# Governed Python Execution on GCP Workstations

## Executive summary

This document describes how to enforce a **single-session, policy-controlled Python runtime** in Google Cloud Workstations.

The approach combines:

- a Python session manager (`workflow_manager.py`),
- a shell shim (`python-wrapper`),
- immutable image packaging,
- startup-time VS Code settings enforcement,
- and strict removal of `sudo`.

The result is a tamper-resistant gatekeeper model that improves:

1. **Exclusivity** (one workflow at a time),
2. **Context integrity** (required environment is injected before execution),
3. **Auditability** (all execution is funneled through one controlled path).

---

## 1) Why this is needed

Cloud IDE hosting alone does not guarantee process governance. In regulated or high-risk environments, Python execution must be mediated so users cannot run arbitrary interpreters outside approved controls.

The requirement is: **all Python execution must pass through an initialization and lock layer**.

---

## 2) GCP Workstations architecture constraints

### 2.1 Split-state storage model

GCP Workstations behaves as two layers:

1. **Immutable container filesystem** (`/usr`, `/etc`, `/opt`)
   - Anything modified at runtime is lost on restart.
   - Governance binaries must be installed here at image build time.

2. **Persistent home mount** (`/home/user`)
   - User data and IDE settings persist.
   - This mount overlays any `/home` content added during image build.

### 2.2 Practical enforcement implication

Do **not** rely on Dockerfile-copied `/home/user/.../settings.json` for policy. It will be hidden by the persistent disk mount.

Use `/etc/workstation-startup.d/` scripts to inject or heal settings at boot.

### 2.3 Trust boundary

If users have `sudo`, local controls are bypassable.

Set this environment variable on the workstation config:

- `CLOUD_WORKSTATIONS_CONFIG_DISABLE_SUDO=true`

Without this, tamper-proofing is not defensible.

---

## 3) Locking model

### 3.1 Advisory locking with `fcntl`

Use file descriptor locking (`fcntl.lockf` or `fcntl.flock`) on a lock file (for example in `/tmp`).

Why this model:

- Avoids stale lock files from crash scenarios.
- Locks are released automatically when the process exits and descriptors close.

### 3.2 Gatekeeper pattern

All interpreter calls are routed through a wrapper that:

1. acquires lock,
2. performs mandatory pre-init,
3. forwards args to real Python,
4. releases lock on exit.

To make this tamper-resistant:

- install in `/usr/local/bin`,
- owner `root:root`,
- mode `755`,
- disable `sudo`.

---

## 4) Reference implementation

### 4.1 `workflow_manager.py`

```python
#!/usr/bin/env python3
import fcntl
import os
import subprocess
import sys

LOCK_FILE_PATH = "/tmp/gcp_workstation_workflow.lock"
REAL_PYTHON_PATH = "/usr/bin/python3"


def acquire_lock():
    """Acquire an exclusive non-blocking lock; return file handle or None."""
    lock_file = open(LOCK_FILE_PATH, "w")
    try:
        fcntl.lockf(lock_file, fcntl.LOCK_EX | fcntl.LOCK_NB)
        return lock_file
    except OSError:
        lock_file.close()
        return None


def is_introspection_command(args):
    """
    Allow VS Code/Pylance interpreter probes to run without acquiring the lock.
    """
    if len(args) >= 2 and args[0] == "-c":
        probe = args[1]
        return ("sys.version" in probe) or ("sys.prefix" in probe)
    return False


def main():
    args = sys.argv[1:]
    introspection = is_introspection_command(args)

    lock = None
    if not introspection:
        lock = acquire_lock()
        if lock is None:
            print("SESSION MANAGER ERROR: Python workflow is currently locked.", file=sys.stderr)
            print("An active session is already running. Try again later.", file=sys.stderr)
            sys.exit(1)

    # Mandatory pre-initialization hook.
    # Add real env/bootstrap steps here.
    os.environ["WORKFLOW_LOCK_OWNER_PID"] = str(os.getpid())
    os.environ["WORKFLOW_SESSION_STATE"] = "LOCKED"

    cmd = [REAL_PYTHON_PATH] + args

    try:
        exit_code = subprocess.call(cmd)
        sys.exit(exit_code)
    except KeyboardInterrupt:
        sys.exit(130)
    except Exception as exc:
        print(f"SESSION MANAGER CRITICAL FAILURE: {exc}", file=sys.stderr)
        sys.exit(1)
    finally:
        if lock:
            fcntl.lockf(lock, fcntl.LOCK_UN)
            lock.close()


if __name__ == "__main__":
    main()
```

### 4.2 `python-wrapper`

```bash
#!/bin/bash
# /usr/local/bin/python-wrapper

MANAGER_SCRIPT="/usr/local/bin/workflow_manager.py"
exec /usr/bin/python3 "$MANAGER_SCRIPT" "$@"
```

### 4.3 File placement strategy

| Path | Purpose | Persistence | Strategy |
|---|---|---|---|
| `/usr/local/bin/python-wrapper` | Interpreter entrypoint for VS Code | Immutable | Build-time copy + `chown root:root` + `chmod 755` |
| `/usr/local/bin/workflow_manager.py` | Lock + pre-init logic | Immutable | Build-time copy + `chown root:root` + `chmod 755` |
| `/etc/workstation-startup.d/` | Runtime policy scripts | Immutable | Build-time copy |
| `/home/user/` | User workspace + settings | Persistent | Runtime injection only |

---

## 5) Container image setup

```dockerfile
FROM us-central1-docker.pkg.dev/cloud-workstations-images/predefined/code-oss:latest

RUN sudo apt-get update && sudo apt-get install -y \
    python3-venv \
    python3-pip \
    jq \
    wget \
    unzip && \
    sudo rm -rf /var/lib/apt/lists/*

COPY workflow_manager.py /usr/local/bin/workflow_manager.py
COPY python-wrapper /usr/local/bin/python-wrapper

RUN sudo chown root:root /usr/local/bin/workflow_manager.py /usr/local/bin/python-wrapper && \
    sudo chmod 755 /usr/local/bin/workflow_manager.py /usr/local/bin/python-wrapper

COPY 300_enforce_config.sh /etc/workstation-startup.d/300_enforce_config.sh
RUN sudo chown root:root /etc/workstation-startup.d/300_enforce_config.sh && \
    sudo chmod 755 /etc/workstation-startup.d/300_enforce_config.sh
```

---

## 6) Runtime settings enforcement

Use startup scripts to force VS Code machine settings after `/home` is mounted.

### 6.1 `300_enforce_config.sh`

```bash
#!/bin/bash
# /etc/workstation-startup.d/300_enforce_config.sh

set -euo pipefail

CODE_DATA_DIR="/home/user/.codeoss-cloudworkstations/data/Machine"
SETTINGS_FILE="$CODE_DATA_DIR/settings.json"
WRAPPER_PATH="/usr/local/bin/python-wrapper"
TMP_MANDATORY="/tmp/mandatory_settings.json"

mkdir -p "$CODE_DATA_DIR"

cat > "$TMP_MANDATORY" <<EOF
{
  "python.defaultInterpreterPath": "$WRAPPER_PATH",
  "python.terminal.activateEnvironment": false,
  "python.experiments.enabled": false,
  "python.analysis.typeCheckingMode": "off"
}
EOF

if command -v jq >/dev/null 2>&1; then
  if [ -f "$SETTINGS_FILE" ] && jq empty "$SETTINGS_FILE" >/dev/null 2>&1; then
    jq -s '.[0] * .[1]' "$SETTINGS_FILE" "$TMP_MANDATORY" > "$SETTINGS_FILE.tmp"
    mv "$SETTINGS_FILE.tmp" "$SETTINGS_FILE"
  else
    mv "$TMP_MANDATORY" "$SETTINGS_FILE"
  fi
else
  cp "$TMP_MANDATORY" "$SETTINGS_FILE"
fi

chown -R user:user "/home/user/.codeoss-cloudworkstations"
```

---

## 7) Multi-mode identity model

For restricted vs privileged workflows, attach **different service accounts** to different workstation configs rather than using human user identity.

Example pattern:

1. `sa-workstation-restricted@...` (read-only scopes)
2. `sa-workstation-privileged@...` (write scopes)
3. Two workstation configs, same image, different service accounts
4. Optional config env var: `WORKFLOW_MODE=RESTRICTED` or `WORKFLOW_MODE=PRIVILEGED`

Your Python code can branch behavior using:

```python
mode = os.environ.get("WORKFLOW_MODE", "RESTRICTED")
```

---

## 8) Deployment checklist

### 8.1 Build image

```yaml
# cloudbuild.yaml
steps:
  - name: 'gcr.io/kaniko-project/executor:latest'
    args:
      - --destination=us-central1-docker.pkg.dev/$PROJECT_ID/workstations-repo/locked-python:v1
      - --cache=true
```

### 8.2 Update workstation config

```bash
gcloud workstations configs update config-restricted-mode \
  --region=us-central1 \
  --cluster=my-cluster \
  --service-account=sa-workstation-restricted@$PROJECT_ID.iam.gserviceaccount.com \
  --container-custom-image=us-central1-docker.pkg.dev/$PROJECT_ID/workstations-repo/locked-python:v1 \
  --env-variables=CLOUD_WORKSTATIONS_CONFIG_DISABLE_SUDO=true,WORKFLOW_MODE=RESTRICTED
```

### 8.3 Validation tests

1. **Tamper test**: editing `/usr/local/bin/python-wrapper` fails; `sudo -i` fails.
2. **Identity test**: `google.auth.default()` resolves to workstation service account.
3. **Lock test**: second concurrent `python-wrapper` call returns lock error.

---

## 9) Key conclusion

This design is effective only when all three conditions are true:

1. Python calls are forced through the wrapper,
2. runtime settings are enforced at startup,
3. `sudo` is disabled.

With those in place, GCP Workstations can operate as a controlled, policy-compliant development appliance.

## Source

1. Customize your development environment | Cloud Workstations: https://docs.cloud.google.com/workstations/docs/customize-development-environment