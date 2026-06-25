# Application Health Check Document
## Elevator Integration Product

---

**Document Version:** 1.0
**Classification:** Internal / Confidential
**Prepared by:** HCL Technologies
**Client:** Amano Corporation
**Date:** June 2025

---

## Table of Contents

1. [Purpose and Scope](#1-purpose-and-scope)
2. [Infrastructure Layer — AWS ECS Fargate](#2-infrastructure-layer--aws-ecs-fargate)
3. [Application Layer — serv1 and serv2](#3-application-layer--serv1-and-serv2)
4. [MQTT Backend Service](#4-mqtt-backend-service)
5. [API Layer — ELV Connection APIs](#5-api-layer--elv-connection-apis)
6. [Database Layer — MySQL](#6-database-layer--mysql)
7. [AWS IoT — Device Shadow](#7-aws-iot--device-shadow)
8. [End-to-End Elevator Flow Validation](#8-end-to-end-elevator-flow-validation)
9. [Monitoring — CloudWatch and ECS Events](#9-monitoring--cloudwatch-and-ecs-events)
10. [Overall Health Status Summary](#10-overall-health-status-summary)

---

## 1. Purpose and Scope

### 1.1 Background

The Elevator Integration Product enables autonomous mobile robots (AMRs) to interact with building elevator systems without human intervention. As of this writing, there is no centralized monitoring tool available for this system. After deployment, new team members and support engineers do not have a standard procedure to verify application health across all layers.

This document addresses that gap. It provides a complete, self-contained health check runbook that any engineer — regardless of prior familiarity with the system — can follow to determine whether every component is operating normally.

### 1.2 Intended Audience

- New team members onboarding to the Elevator Integration Product
- Support engineers responding to incidents or alerts
- On-call engineers performing post-deployment verification
- QA engineers conducting pre-release sanity checks

### 1.3 Components Covered

| Layer | Component | Description |
|---|---|---|
| Infrastructure | AWS ECS Fargate | Container orchestration; cluster and task health |
| Application | serv1, serv2 | Core backend services for elevator command processing |
| Messaging | MQTT Backend | Real-time robot-to-elevator communication broker |
| API | ELV Connection APIs | Registration, Call, Status, Release endpoints |
| Database | MySQL | Persistent storage for sessions and registrations |
| IoT | AWS Device Shadow | Cloud-side state mirror for elevator device |
| Integration | End-to-End Flow | Full robot lifecycle from registration to release |
| Observability | CloudWatch / ECS Events | Log monitoring and container event tracking |

### 1.4 How to Use This Document

Work through sections 2 to 9 in order. Each section follows the same structure:

1. **Overview** — what is being checked and why it matters
2. **Pre-conditions** — what must be true before running checks in this section
3. **Verification Procedure** — step-by-step instructions
4. **Sample Commands** — exact commands to run with placeholders for environment values
5. **Expected Responses** — what a healthy system returns
6. **Validation Criteria** — OK / NG decision table
7. **Failure Scenarios and Recovery** — what to do when something is wrong

For each check, record **OK** if the result matches the expected response, or **NG** if it does not. If any check returns NG, follow the recovery steps in that section before moving to the next section.

### 1.5 Placeholder Reference

All commands in this document use the following placeholders. Replace them with actual values for the target environment before executing.

| Placeholder | Description |
|---|---|
| `<CLUSTER_NAME>` | AWS ECS cluster name |
| `<SERVICE_NAME>` | ECS service name |
| `<TASK_ARN>` | ECS task ARN (retrieved from describe-services output) |
| `<serv1-host>` | Hostname or IP of serv1 |
| `<serv2-host>` | Hostname or IP of serv2 |
| `<port>` | Port number of the service |
| `<BROKER_HOST>` | MQTT broker hostname or IP |
| `<USER>` / `<PASS>` | MQTT broker credentials |
| `<host>` | API gateway or service host |
| `<DB_HOST>` | MySQL hostname or RDS endpoint |
| `<DB_USER>` / `<DB_PASS>` | MySQL credentials |
| `<DB_NAME>` | Target database name |
| `<ELEVATOR_THING_NAME>` | AWS IoT thing name for the elevator device |

---

## 2. Infrastructure Layer — AWS ECS Fargate

### 2.1 Overview

The Elevator Integration Product runs as containerized services on AWS ECS Fargate. This layer is the foundation of the entire system. If ECS tasks are not running and healthy, no application-level check will succeed. This section must be verified first and must pass completely before proceeding to Section 3.

ECS Fargate manages the lifecycle of application containers including serv1 and serv2. The checks in this section confirm that the cluster is active, the correct number of tasks are running, no tasks are stuck in a pending state, and the container health checks configured in the task definition are passing.

### 2.2 Pre-conditions

- AWS CLI is installed and configured with appropriate IAM permissions
- The operator has `ecs:DescribeServices` and `ecs:DescribeTasks` permissions
- The cluster name and service name are known

### 2.3 Verification Procedure

**Step 1 — Describe the ECS service to check cluster and task status**

Run the following command and note the values of `status`, `runningCount`, `desiredCount`, and `pendingCount` in the response.

```bash
aws ecs describe-services \
  --cluster <CLUSTER_NAME> \
  --services <SERVICE_NAME> \
  --query 'services[0].{Status:status,Running:runningCount,Desired:desiredCount,Pending:pendingCount,Events:events[:3]}' \
  --output json
```

**Step 2 — List running task ARNs**

```bash
aws ecs list-tasks \
  --cluster <CLUSTER_NAME> \
  --service-name <SERVICE_NAME> \
  --desired-status RUNNING
```

**Step 3 — Describe individual tasks to check container health**

Use the task ARNs returned in Step 2.

```bash
aws ecs describe-tasks \
  --cluster <CLUSTER_NAME> \
  --tasks <TASK_ARN> \
  --query 'tasks[0].{Health:healthStatus,LastStatus:lastStatus,Containers:containers[*].{Name:name,Health:healthStatus,Status:lastStatus}}' \
  --output json
```

**Step 4 — Review recent ECS service events for anomalies**

```bash
aws ecs describe-services \
  --cluster <CLUSTER_NAME> \
  --services <SERVICE_NAME> \
  --query 'services[0].events[:10]' \
  --output table
```

### 2.4 Expected Responses

**describe-services (healthy state):**

```json
{
  "Status": "ACTIVE",
  "Running": 2,
  "Desired": 2,
  "Pending": 0,
  "Events": [
    {
      "id": "abc123",
      "createdAt": "2025-06-01T10:00:00Z",
      "message": "service <SERVICE_NAME> has reached a steady state."
    }
  ]
}
```

**describe-tasks (healthy state):**

```json
{
  "Health": "HEALTHY",
  "LastStatus": "RUNNING",
  "Containers": [
    {
      "Name": "serv1",
      "Health": "HEALTHY",
      "Status": "RUNNING"
    },
    {
      "Name": "serv2",
      "Health": "HEALTHY",
      "Status": "RUNNING"
    }
  ]
}
```

### 2.5 Validation Criteria

| Check Item | Command | Expected Result | Status |
|---|---|---|---|
| Cluster / Service status | `describe-services .status` | `ACTIVE` | OK / NG |
| Running tasks equal desired count | `describe-services .runningCount` | Equal to `desiredCount` | OK / NG |
| Pending tasks | `describe-services .pendingCount` | `0` | OK / NG |
| Task last status | `describe-tasks .lastStatus` | `RUNNING` | OK / NG |
| Task health status | `describe-tasks .healthStatus` | `HEALTHY` | OK / NG |
| Container health | `describe-tasks .containers[*].healthStatus` | `HEALTHY` for all containers | OK / NG |
| Service events | `describe-services .events` | Latest event: "reached a steady state" | OK / NG |

### 2.6 Failure Scenarios and Recovery

| Symptom | Likely Cause | Recovery Steps |
|---|---|---|
| `runningCount` < `desiredCount` | Task crashed, OOM kill, or startup failure | 1. Check CloudWatch logs for the failing task. 2. Look for exit code 137 (OOM) or application exception. 3. If OOM, increase task memory in task definition. 4. Force redeployment: `aws ecs update-service --cluster <CLUSTER_NAME> --service <SERVICE_NAME> --force-new-deployment` |
| `pendingCount` > 0 for more than 5 minutes | Insufficient cluster capacity, ECR image pull failure, or VPC networking issue | 1. Check service events for error messages. 2. Verify ECR image tag exists: `aws ecr describe-images --repository-name <REPO>`. 3. Confirm task IAM role has `ecr:GetAuthorizationToken` permission. 4. Check VPC DNS resolution and NAT gateway for Fargate networking. |
| `healthStatus: UNHEALTHY` | Application health check endpoint not responding within configured timeout | 1. Retrieve health check definition: `aws ecs describe-task-definition --task-definition <TASK_DEF>`. 2. Manually invoke health endpoint from within the VPC or via bastion. 3. Check application logs for startup errors. 4. Increase health check grace period if service is slow to start. |
| Service events show repeated task replacements | Recurring crash loop | 1. Retrieve stopped task details: `aws ecs list-tasks --cluster <CLUSTER_NAME> --desired-status STOPPED`. 2. Describe stopped task for stop reason and exit code. 3. Tail logs: `aws logs tail /ecs/<SERVICE_NAME> --since 30m`. |

---

## 3. Application Layer — serv1 and serv2

### 3.1 Overview

`serv1` and `serv2` are the core application services of the Elevator Integration Product. They expose HTTP health endpoints that report the service's own status as well as the status of their downstream dependencies (database, MQTT broker). A response of `{ "status": "UP" }` indicates that the service itself is running and all dependencies it is aware of are reachable.

Both services must return `HTTP 200` with `status: UP` before any API-level or integration checks are meaningful.

### 3.2 Pre-conditions

- ECS Fargate checks in Section 2 have passed
- The hostnames or ALB DNS names for serv1 and serv2 are available
- Network access to the services exists (either within VPC, via bastion, or via load balancer)

### 3.3 Verification Procedure

**Step 1 — Check serv1 health endpoint**

```bash
curl -s -o /dev/null -w "HTTP Status: %{http_code}\nResponse Time: %{time_total}s\n" \
  http://<serv1-host>:<port>/health
```

**Step 2 — Check serv1 response body**

```bash
curl -s http://<serv1-host>:<port>/health | jq .
```

**Step 3 — Check serv2 health endpoint**

```bash
curl -s -o /dev/null -w "HTTP Status: %{http_code}\nResponse Time: %{time_total}s\n" \
  http://<serv2-host>:<port>/health
```

**Step 4 — Check serv2 response body**

```bash
curl -s http://<serv2-host>:<port>/health | jq .
```

**Step 5 — Confirm dependency health is also reported as UP (if supported by service)**

```bash
curl -s http://<serv1-host>:<port>/health/details | jq .
```

### 3.4 Expected Responses

**Minimal healthy response:**

```json
{
  "status": "UP"
}
```

**Extended healthy response (if /health/details is available):**

```json
{
  "status": "UP",
  "timestamp": "2025-06-01T10:00:00Z",
  "version": "1.4.2",
  "dependencies": {
    "database": "UP",
    "mqtt": "UP"
  }
}
```

**curl timing output (healthy):**

```
HTTP Status: 200
Response Time: 0.043s
```

> Response time above 2 seconds on the health endpoint is a warning sign even if the status is UP. Note it in the remarks column.

### 3.5 Validation Criteria

| Check Item | Command | Expected Result | Status |
|---|---|---|---|
| serv1 HTTP status code | `curl -w "%{http_code}"` | `200` | OK / NG |
| serv1 response body | `curl \| jq .status` | `"UP"` | OK / NG |
| serv1 response time | `curl -w "%{time_total}"` | Less than 2 seconds | OK / NG |
| serv2 HTTP status code | `curl -w "%{http_code}"` | `200` | OK / NG |
| serv2 response body | `curl \| jq .status` | `"UP"` | OK / NG |
| serv2 response time | `curl -w "%{time_total}"` | Less than 2 seconds | OK / NG |

### 3.6 Failure Scenarios and Recovery

| Symptom | Likely Cause | Recovery Steps |
|---|---|---|
| `curl: (7) Failed to connect` | Service not listening on the expected port, or security group blocking access | 1. Confirm task is RUNNING (Section 2). 2. Verify security group allows inbound traffic on the service port from your source. 3. Confirm port binding in task definition matches the port you are calling. |
| HTTP 503 Service Unavailable | Service is starting up, overloaded, or load balancer has no healthy targets | 1. Wait 60 seconds and retry. 2. If persistent, check ALB target group health in AWS Console. 3. Force redeployment if needed. |
| `{ "status": "DOWN" }` | A downstream dependency (database or MQTT) is unreachable from the service | 1. Do not retry this service. 2. Proceed immediately to Section 4 (MQTT) and Section 6 (Database). 3. Resolve the upstream failure and recheck this endpoint. |
| HTTP 200 but response time > 2s | Service is under load or database queries are slow | 1. Check CloudWatch metrics for CPU and memory usage on the ECS task. 2. Check DB slow query logs. 3. Note in remarks; escalate if sustained. |
| HTTP 404 | Wrong endpoint path or service version mismatch | 1. Confirm the health endpoint path in the service documentation. 2. Verify the deployed image version matches what is expected. |

---

## 4. MQTT Backend Service

### 4.1 Overview

MQTT is the real-time messaging backbone between robots and the elevator controller. The MQTT broker handles command publishing (e.g., "call elevator to floor 3") and status subscriptions (e.g., "robot has boarded"). If the broker is unreachable or authentication fails, the entire robot-elevator communication chain breaks, and the end-to-end flow in Section 8 will fail regardless of API and database health.

This section verifies three things: broker TCP connectivity, credential acceptance, and the ability to publish a message and receive it on a subscribed topic.

### 4.2 Pre-conditions

- `mosquitto_pub` and `mosquitto_sub` are installed on the test host (`apt install mosquitto-clients` or equivalent)
- MQTT broker hostname, port, username, and password are available
- The application health endpoint for the MQTT service is known

### 4.3 Verification Procedure

**Step 1 — Test raw TCP connectivity to the broker**

```bash
nc -zv <BROKER_HOST> 1883
```

Expected output:

```
Connection to <BROKER_HOST> 1883 port [tcp/*] succeeded!
```

**Step 2 — Publish a test message to the health check topic**

```bash
mosquitto_pub \
  -h <BROKER_HOST> \
  -p 1883 \
  -t test/health \
  -m "ping-$(date +%s)" \
  -u <USER> \
  -P <PASS> \
  -d
```

The `-d` flag enables debug output. Confirm that the output contains `CONNACK` with return code `0` (accepted) and `PUBACK`.

**Step 3 — Subscribe to the same topic and verify message receipt**

Open a second terminal and run the subscriber before publishing:

```bash
mosquitto_sub \
  -h <BROKER_HOST> \
  -p 1883 \
  -t test/health \
  -u <USER> \
  -P <PASS> \
  -C 1 \
  -W 10 \
  -d
```

Then publish from the first terminal (Step 2). The subscriber should print the received message within 10 seconds and exit.

**Step 4 — Check MQTT application service health endpoint**

```bash
curl -s http://<mqtt-service-host>:<port>/health | jq .
```

**Step 5 — Verify TLS if broker uses port 8883**

```bash
mosquitto_pub \
  -h <BROKER_HOST> \
  -p 8883 \
  --cafile /path/to/ca.crt \
  -t test/health \
  -m "tls-ping" \
  -u <USER> \
  -P <PASS>
```

### 4.4 Expected Responses

**mosquitto_pub debug output (healthy):**

```
Client mosq-xxxx sending CONNECT
Client mosq-xxxx received CONNACK (0)
Client mosq-xxxx sending PUBLISH (d0, q0, r0, m1, 'test/health', ... (14 bytes))
Client mosq-xxxx sending DISCONNECT
```

> CONNACK return code `0` = Connection Accepted. Any non-zero code indicates failure.

**mosquitto_sub output (healthy):**

```
Client mosq-yyyy received CONNACK (0)
Client mosq-yyyy sending SUBSCRIBE (Mid: 1, Topic: test/health, QoS: 0)
Client mosq-yyyy received SUBACK
ping-1717200000
Client mosq-yyyy sending DISCONNECT
```

**Application health endpoint (healthy):**

```json
{
  "status": "UP",
  "broker": "connected",
  "timestamp": "2025-06-01T10:00:00Z"
}
```

### 4.5 Validation Criteria

| Check Item | Command | Expected Result | Status |
|---|---|---|---|
| TCP connectivity port 1883 | `nc -zv <BROKER_HOST> 1883` | `succeeded` | OK / NG |
| Authentication accepted | `mosquitto_pub -d` CONNACK code | `CONNACK (0)` | OK / NG |
| Publish completes without error | `mosquitto_pub` exit code | `0` (no error) | OK / NG |
| Subscriber receives message | `mosquitto_sub -C 1 -W 10` | Receives published payload within 10s | OK / NG |
| App health endpoint | `GET /health` | `{ "status": "UP" }` | OK / NG |

### 4.6 Failure Scenarios and Recovery

| Symptom | Likely Cause | Recovery Steps |
|---|---|---|
| `nc`: Connection refused on port 1883 | Broker process not running or security group blocking port | 1. Check broker container or EC2 service: `systemctl status mosquitto`. 2. Verify security group allows inbound TCP on port 1883 from the test host CIDR. 3. Restart broker if not running. |
| `CONNACK (5)` — Connection Refused: Not Authorised | Incorrect username or password, or ACL denying the client | 1. Double-check MQTT credentials against the broker password file or secrets manager entry. 2. Review broker ACL file for entries that may deny this client. 3. Test with a known-good client to isolate credential vs ACL issue. |
| `CONNACK (4)` — Bad username or password | Username/password format issue | 1. Verify no trailing whitespace in credentials. 2. Re-export credentials from secrets manager and retry. |
| Publish succeeds but subscriber does not receive message | Topic ACL preventing subscribe, or QoS mismatch | 1. Check broker ACL for subscribe permission on `test/health`. 2. Confirm subscriber is connected before publisher sends. 3. Try QoS 1: add `-q 1` to both pub and sub commands. |
| App health shows `"broker": "disconnected"` | Application lost connection to broker after startup | 1. Restart the MQTT application service container. 2. Check broker logs for client disconnect events. 3. Verify broker is not at max connection limit. |

---

## 5. API Layer — ELV Connection APIs

### 5.1 Overview

The ELV Connection APIs are the primary interface through which robot control software interacts with the elevator integration system. There are four APIs, each corresponding to a phase of the robot-elevator lifecycle. They must be tested in sequence because each subsequent call depends on state created by the previous one.

| Order | API | Purpose |
|---|---|---|
| 1 | Registration API | Registers the robot with the elevator system for a session |
| 2 | Call Elevator API | Sends a command to call the elevator to the robot's current floor |
| 3 | Robot Status API | Queries the current state of the robot within the session |
| 4 | Release API | Closes the session and releases the elevator for other robots |

### 5.2 Pre-conditions

- Sections 3, 4, and 6 (application, MQTT, database) have all passed
- A valid `robotId` and `buildingId` are available for test use
- The API host and port are accessible from the test environment

### 5.3 Verification Procedure

**Step 1 — Registration API**

Register the robot to initiate a session. Note the session ID or registration token returned in the response body — it is required for subsequent calls.

```bash
curl -s -X POST http://<host>:<port>/api/v1/elevator/register \
  -H "Content-Type: application/json" \
  -d '{
    "robotId": "robot-01",
    "floor": 1,
    "buildingId": "bldg-01"
  }' | jq .
```

**Step 2 — Call Elevator API**

Issue the elevator call command using the registered robot ID.

```bash
curl -s -X POST http://<host>:<port>/api/v1/elevator/call \
  -H "Content-Type: application/json" \
  -d '{
    "robotId": "robot-01",
    "targetFloor": 3,
    "direction": "UP"
  }' | jq .
```

**Step 3 — Robot Status API**

Poll the robot's current status. This should reflect the state transitions triggered by the Call command.

```bash
curl -s http://<host>:<port>/api/v1/robot/status/robot-01 | jq .
```

Repeat this call every 10–15 seconds until the status reflects `WAITING`, `BOARDING`, or `IN_ELEVATOR` depending on elevator response time.

**Step 4 — Release API**

Release the elevator session once the robot has exited.

```bash
curl -s -X POST http://<host>:<port>/api/v1/elevator/release \
  -H "Content-Type: application/json" \
  -d '{
    "robotId": "robot-01",
    "elevatorId": "elv-01"
  }' | jq .
```

**Step 5 — Confirm session is cleared from the database**

```bash
mysql -h <DB_HOST> -u <DB_USER> -p<DB_PASS> -D <DB_NAME> \
  -e "SELECT * FROM robot_sessions WHERE robot_id = 'robot-01';"
```

Expected: 0 rows returned, or a row with `status = 'RELEASED'`.

### 5.4 Expected Responses

**Registration API (success):**

```json
{
  "status": "registered",
  "sessionId": "sess-20250601-001",
  "robotId": "robot-01",
  "floor": 1,
  "timestamp": "2025-06-01T10:00:00Z"
}
```

**Call Elevator API (success):**

```json
{
  "status": "called",
  "robotId": "robot-01",
  "targetFloor": 3,
  "elevatorId": "elv-01",
  "estimatedArrivalSeconds": 30,
  "timestamp": "2025-06-01T10:00:05Z"
}
```

**Robot Status API (success — mid-journey):**

```json
{
  "status": "active",
  "robotState": "WAITING",
  "currentFloor": 1,
  "targetFloor": 3,
  "sessionId": "sess-20250601-001",
  "timestamp": "2025-06-01T10:00:10Z"
}
```

**Release API (success):**

```json
{
  "status": "released",
  "robotId": "robot-01",
  "sessionId": "sess-20250601-001",
  "timestamp": "2025-06-01T10:02:00Z"
}
```

### 5.5 Validation Criteria

| Check Item | Command | Expected Result | Status |
|---|---|---|---|
| Registration API HTTP status | `POST /register` | `200 OK` | OK / NG |
| Registration response body | `.status` field | `"registered"` | OK / NG |
| Registration creates session | DB query on `robot_sessions` | Row present with `status = 'REGISTERED'` | OK / NG |
| Call Elevator API HTTP status | `POST /call` | `200 OK` | OK / NG |
| Call response body | `.status` field | `"called"` | OK / NG |
| Call publishes MQTT message | Monitor MQTT topic | Message received on elevator command topic | OK / NG |
| Robot Status API HTTP status | `GET /status/:id` | `200 OK` | OK / NG |
| Status reflects active session | `.status` field | `"active"` | OK / NG |
| Release API HTTP status | `POST /release` | `200 OK` | OK / NG |
| Release response body | `.status` field | `"released"` | OK / NG |
| Session cleared from DB | `SELECT` from `robot_sessions` | 0 rows or `status = 'RELEASED'` | OK / NG |

### 5.6 Failure Scenarios and Recovery

| Symptom | Likely Cause | Recovery Steps |
|---|---|---|
| Registration returns HTTP 404 | Wrong base URL, API gateway misconfiguration, or service not deployed | 1. Confirm service is deployed and running (Section 3). 2. Check API gateway routing rules or load balancer listener rules. 3. Try calling the internal service URL directly (bypassing ALB) from within the VPC. |
| Registration returns HTTP 500 | Unhandled exception, typically a DB write failure or missing required field | 1. Check CloudWatch logs for the service: `aws logs tail /ecs/<SERVICE_NAME> --since 10m`. 2. Look for stack traces or `ERROR` level log entries. 3. Confirm `buildingId` exists in the reference data table. |
| Call Elevator returns HTTP 500 after successful registration | Robot ID not found in DB (registration transaction not committed) | 1. Run DB check: `SELECT * FROM robot_sessions WHERE robot_id = 'robot-01';`. 2. If missing, check for DB write errors in logs. 3. Retry registration and immediately check DB. |
| Call Elevator succeeds but MQTT message not received | MQTT publish failing silently, or topic ACL blocking | 1. Subscribe to the elevator command topic manually: `mosquitto_sub -t elevator/+/command -h <BROKER_HOST>`. 2. Re-issue the call and check if message arrives. 3. Review service logs for MQTT publish errors. |
| Status API returns HTTP 404 for a registered robot | Robot ID not found in session store | 1. Confirm registration was successful and session ID was returned. 2. Check DB: `SELECT * FROM robot_sessions WHERE robot_id = 'robot-01';`. 3. Re-register if session is missing. |
| Release API returns HTTP 500 | Session already closed, or DB delete failing | 1. Check if session still exists in DB. 2. If already released, the error may be safe to ignore. 3. If session exists and delete fails, check DB permissions: `SHOW GRANTS FOR '<DB_USER>'@'%';`. |
| Session not cleared from DB after release | Release API returned 200 but DB update failed silently | 1. Manually close the session: `UPDATE robot_sessions SET status = 'RELEASED' WHERE robot_id = 'robot-01';`. 2. File a bug against the release endpoint for missing DB commit. |

---

## 6. Database Layer — MySQL

### 6.1 Overview

MySQL stores all persistent state for the Elevator Integration Product, including robot session records, elevator registrations, and audit logs. A failed database check will manifest as HTTP 500 errors from the API layer and `status: DOWN` from the application health endpoints. This section verifies connectivity, authentication, basic read/write capability, and the presence and accessibility of critical tables.

### 6.2 Pre-conditions

- The MySQL client is installed on the test host
- DB host, port, username, password, and database name are available
- Network access to port 3306 (or custom port) exists from the test host

### 6.3 Verification Procedure

**Step 1 — Test basic connectivity and authentication**

```bash
mysql -h <DB_HOST> -P 3306 -u <DB_USER> -p<DB_PASS> -e "SELECT 1 AS connectivity_check;"
```

**Step 2 — Confirm the correct database is accessible**

```bash
mysql -h <DB_HOST> -u <DB_USER> -p<DB_PASS> \
  -e "SHOW DATABASES;" | grep <DB_NAME>
```

**Step 3 — Read test against a critical table**

```bash
mysql -h <DB_HOST> -u <DB_USER> -p<DB_PASS> -D <DB_NAME> \
  -e "SELECT COUNT(*) AS session_count FROM robot_sessions;"
```

**Step 4 — Write test**

```bash
mysql -h <DB_HOST> -u <DB_USER> -p<DB_PASS> -D <DB_NAME> \
  -e "
    INSERT INTO health_check (check_time, status) VALUES (NOW(), 'OK');
    SELECT LAST_INSERT_ID() AS inserted_id;
  "
```

**Step 5 — Verify all critical tables exist and are accessible**

```bash
mysql -h <DB_HOST> -u <DB_USER> -p<DB_PASS> -D <DB_NAME> \
  -e "
    SELECT COUNT(*) AS elevator_registrations FROM elevator_registrations;
    SELECT COUNT(*) AS robot_sessions FROM robot_sessions;
    SELECT COUNT(*) AS elevator_commands FROM elevator_commands;
    SELECT COUNT(*) AS audit_logs FROM audit_logs;
  "
```

**Step 6 — Check for any table locks or long-running queries**

```bash
mysql -h <DB_HOST> -u <DB_USER> -p<DB_PASS> \
  -e "SHOW PROCESSLIST;" | grep -v Sleep
```

### 6.4 Expected Responses

**Step 1 — Connectivity check:**

```
+--------------------+
| connectivity_check |
+--------------------+
|                  1 |
+--------------------+
```

**Step 4 — Write test:**

```
+-------------+
| inserted_id |
+-------------+
|          42 |
+-------------+
```

> `inserted_id` must be greater than 0. A value of 0 indicates the insert did not execute.

**Step 5 — Table access:**

```
+------------------------+
| elevator_registrations |
+------------------------+
|                    157 |
+------------------------+
+-----------------+
| robot_sessions  |
+-----------------+
|              23 |
+-----------------+
```

> Row counts can be any non-negative integer. An error is the NG condition, not a zero count.

### 6.5 Validation Criteria

| Check Item | Command | Expected Result | Status |
|---|---|---|---|
| TCP connectivity and auth | `mysql -e "SELECT 1"` | Returns `1` without error | OK / NG |
| Target database accessible | `SHOW DATABASES` | `<DB_NAME>` appears in list | OK / NG |
| Read test | `SELECT COUNT(*) FROM robot_sessions` | Non-error result; any integer >= 0 | OK / NG |
| Write test | `INSERT INTO health_check` | `LAST_INSERT_ID()` > 0 | OK / NG |
| elevator_registrations table | `SELECT COUNT(*)` | No error | OK / NG |
| robot_sessions table | `SELECT COUNT(*)` | No error | OK / NG |
| elevator_commands table | `SELECT COUNT(*)` | No error | OK / NG |
| audit_logs table | `SELECT COUNT(*)` | No error | OK / NG |
| No blocking queries | `SHOW PROCESSLIST` | No long-running queries (> 30s) | OK / NG |

### 6.6 Failure Scenarios and Recovery

| Symptom | Likely Cause | Recovery Steps |
|---|---|---|
| `ERROR 2003 (HY000): Can't connect to MySQL server` | DB host unreachable or port 3306 blocked by security group | 1. Verify RDS or EC2 security group allows TCP 3306 inbound from your source IP or CIDR. 2. Test with: `nc -zv <DB_HOST> 3306`. 3. Confirm the RDS instance is in Available state in AWS Console. |
| `ERROR 1045 (28000): Access denied for user` | Wrong username or password | 1. Re-verify `DB_USER` and `DB_PASS` environment variables in the service task definition. 2. If using AWS Secrets Manager, confirm the secret has not been rotated without updating the task definition. 3. Test login manually with the exact credentials from the secret. |
| `ERROR 1049 (42000): Unknown database` | `DB_NAME` is incorrect or database not created | 1. Run `SHOW DATABASES;` to list available databases. 2. Confirm `DB_NAME` environment variable matches exactly (case-sensitive on Linux). 3. If database is missing, run the database initialization script. |
| `ERROR 1146 (42S02): Table doesn't exist` | Database migration scripts not run or ran against wrong DB | 1. Confirm you are connected to the correct database: `SELECT DATABASE();`. 2. List tables: `SHOW TABLES;`. 3. Run pending migrations and retry. |
| Write test returns `inserted_id: 0` | User lacks INSERT permission, or table does not have an auto-increment primary key | 1. Check grants: `SHOW GRANTS FOR '<DB_USER>'@'%';`. 2. Confirm the `health_check` table schema includes an auto-increment column. |
| `SHOW PROCESSLIST` shows queries running > 30 seconds | Table lock contention or slow query | 1. Identify the blocking query PID. 2. Assess impact before killing: `KILL <pid>;`. 3. Check for missing indexes on frequently queried columns. 4. Escalate to DBA if lock cannot be safely released. |

---

## 7. AWS IoT — Device Shadow

### 7.1 Overview

The AWS IoT Device Shadow is a cloud-side JSON document that represents the current and desired state of the elevator device. It decouples the elevator firmware from the application services — the application writes desired state to the shadow, and the device updates reported state as it executes commands.

For the health check, we verify that the shadow document exists, both `desired` and `reported` state sections are populated, and the `reported` state timestamp indicates the device has communicated recently. A stale `reported` state (older than 10 minutes under normal operation) indicates the physical device or MQTT bridge may have lost connectivity.

### 7.2 Pre-conditions

- AWS CLI is configured with `iot-data:GetThingShadow` permission
- The elevator thing name is known
- The IoT data endpoint is accessible (check with `aws iot describe-endpoint --endpoint-type iot:Data-ATS`)

### 7.3 Verification Procedure

**Step 1 — Retrieve the current device shadow**

```bash
aws iot-data get-thing-shadow \
  --thing-name <ELEVATOR_THING_NAME> \
  /dev/stdout | jq .
```

**Step 2 — Check the age of the reported state**

Extract the timestamp and compare to current time. A difference greater than 600 seconds (10 minutes) is a warning.

```bash
SHADOW_TS=$(aws iot-data get-thing-shadow \
  --thing-name <ELEVATOR_THING_NAME> \
  /dev/stdout | jq '.metadata.reported | to_entries[0].value.timestamp')

CURRENT_TS=$(date +%s)
AGE=$((CURRENT_TS - SHADOW_TS))
echo "Shadow reported state age: ${AGE} seconds"
```

**Step 3 — Check for a delta (desired state not yet applied by device)**

```bash
aws iot-data get-thing-shadow \
  --thing-name <ELEVATOR_THING_NAME> \
  /dev/stdout | jq '.state.delta // "No delta — desired and reported are in sync"'
```

A response of `"No delta"` indicates the device has applied all desired state commands.

**Step 4 — List all thing shadows (if named shadows are used)**

```bash
aws iot-data list-named-shadows-for-thing \
  --thing-name <ELEVATOR_THING_NAME>
```

### 7.4 Expected Responses

**Full shadow document (healthy state):**

```json
{
  "state": {
    "desired": {
      "elevatorStatus": "idle",
      "targetFloor": 1,
      "doorState": "closed"
    },
    "reported": {
      "elevatorStatus": "idle",
      "currentFloor": 1,
      "doorState": "closed",
      "firmwareVersion": "2.3.1"
    }
  },
  "metadata": {
    "desired": {
      "elevatorStatus": { "timestamp": 1717200000 },
      "targetFloor": { "timestamp": 1717200000 }
    },
    "reported": {
      "elevatorStatus": { "timestamp": 1717200050 },
      "currentFloor": { "timestamp": 1717200050 }
    }
  },
  "version": 142,
  "timestamp": 1717200100
}
```

**Delta response (desired not yet applied — transient, acceptable during active commands):**

```json
{
  "targetFloor": 3,
  "elevatorStatus": "moving"
}
```

> A persistent delta (unchanged for more than 5 minutes outside of an active elevator command) indicates the device is not applying state updates and requires investigation.

**Age check output (healthy):**

```
Shadow reported state age: 47 seconds
```

### 7.5 Validation Criteria

| Check Item | Command | Expected Result | Status |
|---|---|---|---|
| Shadow document exists | `get-thing-shadow` | Returns JSON without error | OK / NG |
| `desired` state present | `.state.desired` | Non-null, non-empty object | OK / NG |
| `reported` state present | `.state.reported` | Non-null, non-empty object | OK / NG |
| `reported` timestamp is recent | Age calculation script | Less than 600 seconds | OK / NG |
| No persistent delta | `.state.delta` | Null or no delta (when no active command) | OK / NG |
| Shadow version incrementing | `.version` | Greater than 0; increases between checks | OK / NG |

### 7.6 Failure Scenarios and Recovery

| Symptom | Likely Cause | Recovery Steps |
|---|---|---|
| `ResourceNotFoundException` | Thing name is incorrect, or thing has not been provisioned in IoT Core | 1. List things: `aws iot list-things`. 2. Confirm `ELEVATOR_THING_NAME` matches exactly. 3. If thing is missing, re-run the device provisioning script. |
| `reported` state is stale (> 10 minutes old) | Device has lost MQTT connectivity to IoT Core | 1. Check MQTT section (Section 4) for broker connectivity. 2. Verify the device-side IoT Core endpoint configuration. 3. Check device firmware logs for MQTT reconnect errors. 4. Confirm IoT Core endpoint: `aws iot describe-endpoint --endpoint-type iot:Data-ATS`. |
| Persistent delta present (> 5 minutes, no active command) | Device connected but not processing shadow updates | 1. Check device firmware logs for shadow update handler errors. 2. Publish a test desired state change and monitor if reported updates within 60 seconds. 3. Restart the device MQTT client if possible. |
| `UnauthorizedException` when calling `get-thing-shadow` | AWS CLI role lacks `iot-data:GetThingShadow` permission | 1. Check IAM policy attached to the CLI role. 2. Add permission: `iot:GetThingShadow` on the specific thing ARN. |

---

## 8. End-to-End Elevator Flow Validation

### 8.1 Overview

This section validates the complete robot-elevator lifecycle from initial registration through final release. It is the most comprehensive check in this document because it exercises all layers simultaneously: API processing, MQTT messaging, database writes, Device Shadow updates, and the physical or simulated elevator response.

All previous sections (2 through 7) must be marked OK before executing this test. A failure at any phase indicates an integration break between layers.

### 8.2 Pre-conditions

- All checks in Sections 2 through 7 have passed
- A test robot ID (`robot-01` or similar) is available and not currently registered in any active session
- A MQTT subscriber is running to monitor elevator command and status topics
- The database is accessible for session state verification between steps

### 8.3 Flow Overview

```
Register --> Call Elevator --> Boarding --> Arrival --> Exit --> Release
   |               |               |           |          |         |
  DB write     MQTT publish    Shadow Δ    Shadow Δ    MQTT msg  DB clear
```

### 8.4 Verification Procedure

**Step 1 — Confirm no active session exists for the test robot**

```bash
mysql -h <DB_HOST> -u <DB_USER> -p<DB_PASS> -D <DB_NAME> \
  -e "SELECT * FROM robot_sessions WHERE robot_id = 'robot-01' AND status != 'RELEASED';"
```

Expected: 0 rows. If rows are returned, release the existing session first (Step 6 of this procedure) before proceeding.

**Step 2 — Open a MQTT subscriber to monitor real-time events**

Run this in a separate terminal and keep it open throughout the test.

```bash
mosquitto_sub \
  -h <BROKER_HOST> \
  -p 1883 \
  -t "elevator/#" \
  -t "robot/#" \
  -u <USER> \
  -P <PASS> \
  -v
```

**Step 3 — Phase 1: Registration**

```bash
curl -s -X POST http://<host>:<port>/api/v1/elevator/register \
  -H "Content-Type: application/json" \
  -d '{"robotId": "robot-01", "floor": 1, "buildingId": "bldg-01"}' | jq .
```

After this step, verify in DB:

```bash
mysql -h <DB_HOST> -u <DB_USER> -p<DB_PASS> -D <DB_NAME> \
  -e "SELECT robot_id, status, created_at FROM robot_sessions WHERE robot_id = 'robot-01';"
```

**Step 4 — Phase 2: Call Elevator**

```bash
curl -s -X POST http://<host>:<port>/api/v1/elevator/call \
  -H "Content-Type: application/json" \
  -d '{"robotId": "robot-01", "targetFloor": 3, "direction": "UP"}' | jq .
```

Monitor the MQTT subscriber terminal. A message should appear on the elevator command topic within 2–3 seconds.

**Step 5 — Phase 3 and 4: Monitor Boarding and Arrival**

Poll the status API every 15 seconds. The `robotState` field should transition through: `WAITING` → `BOARDING` → `IN_ELEVATOR` → `ARRIVED`.

```bash
watch -n 15 "curl -s http://<host>:<port>/api/v1/robot/status/robot-01 | jq '{robotState, currentFloor, targetFloor}'"
```

Also monitor Device Shadow for floor updates:

```bash
aws iot-data get-thing-shadow \
  --thing-name <ELEVATOR_THING_NAME> \
  /dev/stdout | jq '{currentFloor: .state.reported.currentFloor, targetFloor: .state.desired.targetFloor}'
```

**Step 6 — Phase 5 and 6: Exit and Release**

Once the robot has arrived and exited (status shows `EXITED`), release the session:

```bash
curl -s -X POST http://<host>:<port>/api/v1/elevator/release \
  -H "Content-Type: application/json" \
  -d '{"robotId": "robot-01", "elevatorId": "elv-01"}' | jq .
```

Verify session cleared from DB:

```bash
mysql -h <DB_HOST> -u <DB_USER> -p<DB_PASS> -D <DB_NAME> \
  -e "SELECT robot_id, status FROM robot_sessions WHERE robot_id = 'robot-01';"
```

### 8.5 Expected Outcomes Per Phase

| Phase | API Response | DB State | MQTT Event | Shadow State |
|---|---|---|---|---|
| 1 — Register | `"status": "registered"` | Row created, `status = REGISTERED` | — | — |
| 2 — Call | `"status": "called"` | `status = CALLING` | Command published to elevator topic | `desired.targetFloor` updated |
| 3 — Boarding | Status API: `"BOARDING"` | `status = BOARDING` | Boarding event on robot topic | — |
| 4 — Arrival | Status API: `"ARRIVED"` | `status = ARRIVED` | Arrival event on elevator topic | `reported.currentFloor == targetFloor` |
| 5 — Exit | Status API: `"EXITED"` | `status = EXITED` | Exit event on robot topic | — |
| 6 — Release | `"status": "released"` | `status = RELEASED` or row deleted | — | — |

### 8.6 Validation Criteria

| Check Item | Expected Result | Status |
|---|---|---|
| Phase 1: Registration API returns 200 | HTTP 200 + `"registered"` | OK / NG |
| Phase 1: DB session record created | Row present in `robot_sessions` | OK / NG |
| Phase 2: Call API returns 200 | HTTP 200 + `"called"` | OK / NG |
| Phase 2: MQTT command message published | Message received on elevator topic within 5s | OK / NG |
| Phase 3: Status transitions to BOARDING | Status API shows `"BOARDING"` | OK / NG |
| Phase 4: Elevator arrives at target floor | Shadow `reported.currentFloor == targetFloor` | OK / NG |
| Phase 5: Status transitions to EXITED | Status API shows `"EXITED"` | OK / NG |
| Phase 6: Release API returns 200 | HTTP 200 + `"released"` | OK / NG |
| Phase 6: DB session cleared | 0 active rows for `robot-01` | OK / NG |

### 8.7 Failure Scenarios and Recovery

| Symptom | Likely Cause | Recovery Steps |
|---|---|---|
| Phase 1 fails | API or database unavailable | Resolve Section 5 (API) and Section 6 (DB) failures first. Do not proceed. |
| Phase 2 fails — no MQTT message published | MQTT service cannot publish, or topic ACL issue | 1. Check Section 4. 2. Subscribe to `elevator/#` and re-issue the call. 3. Check service logs for MQTT publish errors. |
| Phase 3 stuck — status stays at WAITING | Elevator controller not responding to command | 1. Confirm elevator device is powered and connected. 2. Check Device Shadow for delta. 3. Reissue the call command. |
| Phase 4 fails — shadow floor not updating | Device not sending reported state updates | 1. Check device firmware logs. 2. Confirm device MQTT connectivity to IoT Core. 3. Manually update shadow for testing only: `aws iot-data update-thing-shadow ...`. |
| Phase 6 fails — release returns error | Session not in a releasable state, or DB issue | 1. Check DB session status. 2. If `status != EXITED`, the robot may not have completed exit. Wait and retry. 3. Manual DB cleanup if necessary: `UPDATE robot_sessions SET status = 'RELEASED' WHERE robot_id = 'robot-01';`. |

---

## 9. Monitoring — CloudWatch and ECS Events

### 9.1 Overview

Even when all API and integration checks pass, the system may have accumulated errors, warnings, or container restarts that indicate instability. This section checks CloudWatch log streams and ECS service events to surface issues that do not immediately present as failures but could lead to degraded performance or an outage.

### 9.2 Pre-conditions

- AWS CLI is configured with `logs:FilterLogEvents`, `logs:GetLogEvents`, and `ecs:DescribeServices` permissions
- The log group name for each service is known (typically `/ecs/<SERVICE_NAME>`)

### 9.3 Verification Procedure

**Step 1 — Tail recent logs from the application services**

```bash
aws logs tail /ecs/<SERVICE_NAME> \
  --follow \
  --since 1h \
  --format short
```

Press `Ctrl+C` after reviewing approximately 5 minutes of output.

**Step 2 — Filter for ERROR-level log events**

```bash
aws logs filter-log-events \
  --log-group-name /ecs/<SERVICE_NAME> \
  --filter-pattern "ERROR" \
  --start-time $(date -d '1 hour ago' +%s000) \
  --query 'events[*].{Time:timestamp,Message:message}' \
  --output table
```

**Step 3 — Filter for FATAL or CRITICAL log events**

```bash
aws logs filter-log-events \
  --log-group-name /ecs/<SERVICE_NAME> \
  --filter-pattern "?FATAL ?CRITICAL" \
  --start-time $(date -d '1 hour ago' +%s000) \
  --output table
```

**Step 4 — Check for exception stack traces**

```bash
aws logs filter-log-events \
  --log-group-name /ecs/<SERVICE_NAME> \
  --filter-pattern "Exception" \
  --start-time $(date -d '1 hour ago' +%s000) \
  --output table
```

**Step 5 — Review ECS service events for container restarts or task failures**

```bash
aws ecs describe-services \
  --cluster <CLUSTER_NAME> \
  --services <SERVICE_NAME> \
  --query 'services[0].events[:15]' \
  --output table
```

**Step 6 — Check for stopped tasks and retrieve their stop reason**

```bash
STOPPED_TASKS=$(aws ecs list-tasks \
  --cluster <CLUSTER_NAME> \
  --desired-status STOPPED \
  --query 'taskArns' \
  --output text)

if [ -n "$STOPPED_TASKS" ]; then
  aws ecs describe-tasks \
    --cluster <CLUSTER_NAME> \
    --tasks $STOPPED_TASKS \
    --query 'tasks[*].{TaskArn:taskArn,StopCode:stopCode,StoppedReason:stoppedReason,Containers:containers[*].{Name:name,ExitCode:exitCode,Reason:reason}}' \
    --output json
fi
```

### 9.4 Expected Responses

**Step 2 — ERROR filter (healthy — no results):**

```
-----------------------------------------------
|          FilterLogEvents                    |
+------------------+--------------------------+
|  Time            |  Message                 |
+------------------+--------------------------+
```

> An empty table means no ERROR events in the past hour. This is the expected healthy state.

**Step 5 — ECS service events (healthy):**

```
---------------------------------------------------------------------------------------------
|                              DescribeServices                                             |
+----------------------------+--------------------------------------------------------------+
| 2025-06-01T10:00:00Z       | service <SERVICE_NAME> has reached a steady state.           |
+----------------------------+--------------------------------------------------------------+
```

> Events showing "task stopped" or "unable to consistently start tasks" are NG.

### 9.5 Validation Criteria

| Check Item | Command | Expected Result | Status |
|---|---|---|---|
| No ERROR logs in past 1 hour | `filter-log-events ERROR` | 0 matching events | OK / NG |
| No FATAL or CRITICAL logs | `filter-log-events ?FATAL ?CRITICAL` | 0 matching events | OK / NG |
| No unhandled exceptions | `filter-log-events Exception` | 0 matching events | OK / NG |
| Container restart count | ECS service events | No "task stopped" events in past 1 hour | OK / NG |
| ECS steady state reached | Latest ECS service event message | Contains "reached a steady state" | OK / NG |
| No stopped tasks | `list-tasks --desired-status STOPPED` | Empty list or tasks stopped > 1 hour ago | OK / NG |

### 9.6 Critical Log Patterns Reference

When reviewing raw logs, the following patterns indicate specific failures and should be escalated immediately.

| Log Pattern | Indicates | Immediate Action |
|---|---|---|
| `Connection refused` + DB host | Database unreachable from service | Re-run Section 6. Check DB security group. |
| `MQTT CONNACK failed` | Broker authentication or network failure | Re-run Section 4. Check broker status. |
| `exit code 137` in ECS events | OOM kill — container ran out of memory | Increase task memory in task definition. |
| `Health check grace period expired` | Service failed to start in time | Check startup logs. Increase health check grace period. |
| `java.lang.OutOfMemoryError` or `heap space` | JVM memory exhaustion (if Java-based) | Increase JVM heap flags in task definition environment variables. |
| `too many connections` (MySQL) | DB connection pool exhausted | Check `max_connections` on DB. Reduce pool size in service config or scale DB. |
| `certificate expired` | TLS certificate for MQTT or API has expired | Renew certificate immediately. Check ACM or custom cert rotation schedule. |

---

## 10. Overall Health Status Summary

### 10.1 Status Recording Table

After completing all checks in Sections 2 through 9, record the final status of each component below. Sign and date the completed runbook before sharing or filing.

| Component | Section | Status (OK / NG) | Notes / Remarks |
|---|---|---|---|
| ECS Fargate Infrastructure | Section 2 | | |
| Application Services (serv1, serv2) | Section 3 | | |
| MQTT Backend | Section 4 | | |
| API Layer — ELV Connection APIs | Section 5 | | |
| Database — MySQL | Section 6 | | |
| AWS IoT Device Shadow | Section 7 | | |
| End-to-End Elevator Flow | Section 8 | | |
| Monitoring — CloudWatch / ECS Events | Section 9 | | |
| **OVERALL SYSTEM STATUS** | All | | |

> The system is considered **HEALTHY** only when all components above are marked **OK**. Any single NG blocks the overall status from being marked HEALTHY.

### 10.2 Reference — Expected Healthy System JSON

This JSON structure is the machine-readable equivalent of the table above and can be used by automated monitoring scripts to report system state.

```json
{
  "overall_status": "HEALTHY",
  "checked_at": "2025-06-01T10:30:00Z",
  "checked_by": "<engineer-name>",
  "components": {
    "fargate":        { "status": "OK", "notes": "" },
    "application":    { "status": "OK", "notes": "" },
    "mqtt":           { "status": "OK", "notes": "" },
    "api_layer":      { "status": "OK", "notes": "" },
    "database":       { "status": "OK", "notes": "" },
    "device_shadow":  { "status": "OK", "notes": "" },
    "elevator_flow":  { "status": "OK", "notes": "" },
    "monitoring":     { "status": "OK", "notes": "" }
  }
}
```

### 10.3 Escalation Path

If any component is NG after following the recovery steps in the corresponding section, escalate using the following path:

| Failing Component | Escalate To | Information to Provide |
|---|---|---|
| ECS Fargate | Infrastructure / DevOps lead | Task ARN, stop reason, CloudWatch log link |
| Application services | Application service owner | Health endpoint response, service logs, error code |
| MQTT | Messaging infrastructure owner | CONNACK code, broker host, topic attempted |
| API Layer | Backend development lead | Endpoint URL, request body, HTTP status, response body, CloudWatch logs |
| Database | Database administrator | Error code, query attempted, DB host |
| Device Shadow | IoT / Firmware team | Thing name, shadow age, delta contents |
| End-to-End Flow | Integration lead | Phase at which flow failed, DB session state, MQTT subscriber output |

### 10.4 Document Revision Log

| Version | Date | Author | Changes |
|---|---|---|---|
| 1.0 | June 2025 | HCL Technologies | Initial release |

---

*End of Document — Elevator Integration Product Application Health Check Runbook v1.0*
*Prepared by HCL Technologies for Amano Corporation — Internal Use Only*
