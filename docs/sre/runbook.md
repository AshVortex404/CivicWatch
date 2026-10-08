# CivicWatch Operational Runbook

> **Security Notice**: This runbook contains operational troubleshooting guidance and diagnostic commands for CivicWatch. It strictly excludes secrets, passwords, database URIs, or authentication tokens.

---

## Overview & Quick Reference

This runbook provides step-by-step diagnostic and remediation procedures for common operational alerts and failure scenarios in the CivicWatch platform.

### Primary Diagnostic Tools:
- **Kubernetes CLI**: Target namespaces `civicwatch`, `monitoring`, `logging`.
- **Prometheus Metrics**: Scraped from `http://civicwatch-backend.civicwatch.svc.cluster.local:5000/metrics`.
- **Grafana Dashboards**: SRE Dashboard (`civicwatch-sre-dashboard`), Observability Dashboard (`civicwatch-observability`), Centralized Logs (`civicwatch-logs`).
- **Centralized Logging**: Loki queries via Grafana or LogQL.

---

## Scenario 1: Backend Unavailable (`BackendDown` Alert)

### Trigger Condition:
Prometheus alert `BackendDown` fires (`up{job="civicwatch-backend"} == 0` for 2+ minutes).

### Diagnostic Steps:
1. **Check Pod Status in Kubernetes**:
   ```bash
   kubectl get pods -n civicwatch -l app=civicwatch-backend
   ```
2. **Inspect Pod Events and Failure Rationale**:
   ```bash
   kubectl describe pod -l app=civicwatch-backend -n civicwatch
   ```
   *Look for: OOMKilled, CrashLoopBackOff, ImagePullBackOff, or failed Liveness/Readiness probes (`GET /health`).*

3. **Check Container Logs via kubectl**:
   ```bash
   kubectl logs -n civicwatch -l app=civicwatch-backend --tail=100
   ```

4. **Check Centralized Loki Logs**:
   In Grafana Loki Explore:
   ```logql
   {namespace="civicwatch", app="civicwatch-backend"} |= "uncaughtException"
   ```

### Remediation Steps:
- **If Pod is CrashLooping**: Check recent deployment history and roll back to last stable revision:
  ```bash
  kubectl rollout undo deployment/civicwatch-backend -n civicwatch
  ```
- **If Pod is Deadlocked or OOMKilled**: Restart the deployment:
  ```bash
  kubectl rollout restart deployment/civicwatch-backend -n civicwatch
  ```
- **Verify Recovery**: Confirm scrape target is back UP:
  PromQL: `up{job="civicwatch-backend"}` (Expected value: `1`).

---

## Scenario 2: High 5xx Error Rate (`HighErrorRate` Alert)

### Trigger Condition:
Prometheus alert `HighErrorRate` fires (`5xx error rate > 1%` for 5+ minutes).

### Diagnostic Steps:
1. **Inspect Status Code Distribution in Grafana**:
   PromQL query for status code breakdown:
   ```promql
   sum(rate(http_requests_total[5m])) by (status_code, route)
   ```
2. **Query Application Exception Logs in Loki**:
   ```logql
   {namespace="civicwatch", app="civicwatch-backend"} |~ "500|Error|Exception"
   ```
3. **Verify Database Connection Errors**:
   Look for MongoDB disconnection errors or query timeouts in logs:
   ```logql
   {namespace="civicwatch", app="civicwatch-backend"} |= "MongoNetworkError"
   ```

### Remediation Steps:
- **If Caused by Bad Code Deploy**: Roll back deployment:
  ```bash
  kubectl rollout undo deployment/civicwatch-backend -n civicwatch
  ```
- **If Database Connection Failure**: Verify MongoDB pod or external URI access (see Scenario 5).
- **Verify Recovery**: Confirm 5xx rate drops below 1%:
  ```promql
  sum(rate(http_requests_total{status_code=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))
  ```

---

## Scenario 3: High Latency (`HighLatency` Alert)

### Trigger Condition:
Prometheus alert `HighLatency` fires (`p95 latency > 500ms` for 5+ minutes).

### Diagnostic Steps:
1. **Evaluate Latency Distribution**:
   ```promql
   histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, route))
   ```
   *Identify specific slow routes (e.g. `/api/issues`, `/api/auth`).*

2. **Check Active Request Concurrency**:
   ```promql
   http_active_requests
   ```
3. **Inspect CPU & Memory Metrics**:
   ```promql
   nodejs_heap_size_used_bytes
   process_cpu_seconds_total
   ```

### Remediation Steps:
- **If Resource Constrained**: Scale backend replicas to distribute load:
  ```bash
  kubectl scale deployment/civicwatch-backend --replicas=3 -n civicwatch
  ```
- **If Event Loop Blocked**: Investigate unindexed MongoDB queries or synchronous loop operations in application code.

---

## Scenario 4: Frontend Service Unavailable

### Trigger Condition:
Users report UI blank page / timeout, or frontend HTTP 502/504 errors.

### Diagnostic Steps:
1. **Check Frontend Pod Status**:
   ```bash
   kubectl get pods -n civicwatch -l app=civicwatch-frontend
   ```
2. **Inspect Nginx / Web Server Logs**:
   ```bash
   kubectl logs -n civicwatch -l app=civicwatch-frontend --tail=100
   ```
3. **Check Frontend Service & Ingress/Port Binding**:
   ```bash
   kubectl get svc civicwatch-frontend -n civicwatch
   ```

### Remediation Steps:
- **Restart Frontend Deployment**:
  ```bash
  kubectl rollout restart deployment/civicwatch-frontend -n civicwatch
  ```
- **Verify Static Asset Serving**: Ensure Nginx configuration correctly proxies `/api` requests to `civicwatch-backend.civicwatch.svc.cluster.local:5000`.

---

## Scenario 5: MongoDB Connectivity Issue

### Trigger Condition:
Backend logs report `MongooseServerSelectionError` or database query timeouts.

### Diagnostic Steps:
1. **Check Backend Database Log Outputs in Loki**:
   ```logql
   {namespace="civicwatch", app="civicwatch-backend"} |~ "Mongoose|Mongo|connection|timeout"
   ```
2. **Verify Environment Variable secret reference (Without Exposing Values)**:
   ```bash
   kubectl describe secret civicwatch-secrets -n civicwatch
   ```
3. **Test Network Connectivity from Backend Pod**:
   ```bash
   kubectl exec -it deployment/civicwatch-backend -n civicwatch -- nc -zv mongodb.civicwatch.svc.cluster.local 27017
   ```

### Remediation Steps:
- **If Database Container / Service Down**: Restart database pod or service.
- **If Network Policy Blocking**: Verify Kubernetes NetworkPolicy rules allow port `27017` egress from `civicwatch-backend` to database endpoints.
- **Verify Recovery**: Backend log indicates `MongoDB Connected Successfully` upon reconnection retry.
