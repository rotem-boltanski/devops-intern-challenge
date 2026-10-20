# Root Cause Analysis

Fill this in as you go. Bullet points are fine — be specific, not long.

---

## Defect 1: Proxy image fails to build

### Symptom / error output
* `docker compose build` failed during the `proxy` image build step.
* Output error: `E: Sub-process /usr/bin/dpkg returned an error code (1)` with `end of file on stdin at conffile prompt`.

### How I diagnosed it
* Ran `./scripts/run-tests.sh` and inspected the complete build output log for the `proxy` container instead of just reading the final error line.

### Root cause
* In `proxy/Dockerfile`, `COPY nginx.conf /etc/nginx/nginx.conf` was executed before `apt-get install -y nginx`.
* Installing `nginx` afterwards created an interactive prompt asking whether to overwrite the pre-existing configuration file, causing `apt-get` to fail in non-interactive Docker build mode.

### Fix
* Reordered `proxy/Dockerfile` to run `apt-get update && apt-get install` before copying `nginx.conf`.
* Added `ENV DEBIAN_FRONTEND=noninteractive` to prevent future interactive prompts.

---

## Defect 2: Proxy cannot reach the backend

### Symptom / error output
* Integration test failed on `GET /api/info` with `HTTP Error 502: Bad Gateway`.
* Proxy log showed: `connect() failed (111: Connection refused) while connecting to upstream ... upstream: "http://127.0.0.1:8080/api/info"`.

### How I diagnosed it
* Executed `./scripts/run-tests.sh` and inspected the Nginx reverse-proxy logs (`proxy-1`).

### Root cause
* In `proxy/nginx.conf`, `proxy_pass` was set to `http://127.0.0.1:8080`.
* In a multi-container Docker Compose setup, `127.0.0.1` refers to the `proxy` container's loopback interface rather than the `backend` service container.

### Fix
* Updated `proxy_pass` in `proxy/nginx.conf` to target `http://backend:8080`, using Docker Compose internal network DNS resolution.

---

## Defect 3: Tests still fail after fixing defect 2

### Symptom / error output
* Integration test suite crashed with `HTTP Error 502: Bad Gateway` because `test-runner` executed queries before `backend` was fully listening on port 8080.
* Additionally, using `curl` for the Docker healthcheck failed because the `python:3.11-alpine` base image lacks pre-installed `curl`.

### How I diagnosed it
* Analyzed startup log timestamps in `./scripts/run-tests.sh` and observed `Backend listening on port 8080...` printed after `test-runner` failed.

### Root cause
* Race condition due to Docker Compose starting containers concurrently without verifying application-level service readiness.

### Fix, and why I chose it over the alternatives
* Added a Python-native `healthcheck` (`urllib.request`) to the `backend` service in `docker-compose.yml` and enforced `depends_on` with `condition: service_healthy` for dependent services.
* **Why this fix**: Native Docker healthchecks eliminate race conditions cleanly without adding unneeded OS packages to minimal Alpine images or resorting to arbitrary `sleep` delays.

---

## Defect 4: CI pipeline fails

### Symptom / error output

### How I diagnosed it

### Root cause

### Fix

---

## Bonus questions

Optional. Answer only the ones you got to — number them, skip the rest.

>

---

## Anything you would do differently with more time