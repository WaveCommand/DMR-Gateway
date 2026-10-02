# DMR Gateway

**Project Codename:** `dmr-gateway`  
**Specification Version:** `0.2.0`  
**Document Status:** Architecture & Product Specification  
**Last Updated:** 2026-10-01  
**Target:** Raspberry Pi / Linux / Docker  
**License:** TBD

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Vision](#2-vision)
3. [Problem Statement](#3-problem-statement)
4. [Design Principles](#4-design-principles)
6. [Goals](#6-goals)
7. [Non-Goals](#7-non-goals)
8. [Core Concepts](#8-core-concepts)
9. [System Architecture](#9-system-architecture)
10. [Trust Boundaries](#10-trust-boundaries)
11. [Event-Driven Architecture](#11-event-driven-architecture)
12. [Event Model](#12-event-model)
13. [Device Abstraction](#13-device-abstraction)
14. [DMR Backend Architecture](#14-dmr-backend-architecture)
15. [MMDVM Integration](#15-mmdvm-integration)
16. [Motorola IPSC Integration](#16-motorola-ipsc-integration)
17. [Hytera Integration](#17-hytera-integration)
18. [Rules Engine](#18-rules-engine)
19. [Conditions](#19-conditions)
20. [Actions](#20-actions)
21. [Workflows](#21-workflows)
22. [Task and Job System](#22-task-and-job-system)
23. [Alert System](#23-alert-system)
24. [DMR Command System](#24-dmr-command-system)
25. [Radio Identity and Authorization](#25-radio-identity-and-authorization)
26. [TTS Architecture](#26-tts-architecture)
27. [MQTT and Home Assistant](#27-mqtt-and-home-assistant)
28. [Cellular SMS Gateway](#28-cellular-sms-gateway)
29. [Hardware Agent](#29-hardware-agent)
30. [Edge Functions](#30-edge-functions)
31. [Secrets Management](#31-secrets-management)
32. [Authentication and Authorization](#32-authentication-and-authorization)
33. [Security Architecture](#33-security-architecture)
34. [Audit Logging](#34-audit-logging)
35. [Observability](#35-observability)
36. [Event Explorer](#36-event-explorer)
37. [Simulation and Dry-Run](#37-simulation-and-dry-run)
38. [Event Replay](#38-event-replay)
39. [Web Application](#39-web-application)
40. [CLI](#40-cli)
41. [Database Model](#41-database-model)
42. [API Design](#42-api-design)
43. [WebSocket Design](#43-websocket-design)
44. [Internal Service Contracts](#44-internal-service-contracts)
45. [Configuration](#45-configuration)
46. [Docker Architecture](#46-docker-architecture)
47. [Networking](#47-networking)
48. [Persistence](#48-persistence)
49. [Reliability](#49-reliability)
50. [Failure Handling](#50-failure-handling)
51. [Rate Limiting](#51-rate-limiting)
52. [Validation](#52-validation)
53. [Testing Strategy](#53-testing-strategy)
54. [Developer Experience](#54-developer-experience)
55. [Repository Structure](#55-repository-structure)
56. [Technology Stack](#56-technology-stack)
57. [Development Phases](#57-development-phases)
58. [Milestones](#58-milestones)
59. [Operational Runbook](#59-operational-runbook)
60. [Backup and Recovery](#60-backup-and-recovery)
61. [Upgrade Strategy](#61-upgrade-strategy)
62. [Performance Targets](#62-performance-targets)
63. [Future Features](#63-future-features)
64. [Example Automations](#64-example-automations)
65. [Example API Requests](#65-example-api-requests)
66. [Example Event Payloads](#66-example-event-payloads)
67. [Architectural Decisions](#67-architectural-decisions)
68. [Open Questions](#68-open-questions)
69. [Definition of Done](#69-definition-of-done)
70. [Conclusion](#70-conclusion)

---

# 1. Project Overview

DMR Gateway is a self-hosted, Docker-based, event-driven automation platform designed to connect DMR radio infrastructure with physical hardware, messaging systems, home automation platforms, web services, cellular networks, and user-defined automation.

The initial target platform is a Raspberry Pi or another Linux host, but the architecture is intentionally designed so that the same software can run on x86_64 servers, ARM64 systems, virtual machines, and larger dedicated infrastructure.

The platform treats DMR as both:

- an **event source**, and
- an **action destination**.

This distinction is fundamental.

A DMR transmission can generate an event:

```text
DMR SMS received
DMR PTT started
DMR PTT ended
DMR voice transmission started
DMR voice transmission ended
Radio registration changed
Talkgroup activity detected
```

Those events can then be processed by the automation engine.

Conversely, automation can produce DMR actions:

```text
Send DMR SMS
Broadcast TTS audio
Trigger a DMR alert
Send a confirmation
```

The system is not limited to DMR. Other event sources and action targets include:

```text
MQTT
HTTP
GPIO
Serial
Cellular SMS
Schedules
Webhooks
Cameras
Home Assistant
Edge Functions
```

The long-term objective is to provide a unified automation layer where radio activity can participate in the same automation model as IoT and software events.

---

# 2. Vision

The project should evolve into:

> **A self-hosted event-driven automation platform for DMR networks and connected infrastructure.**

The platform should allow an operator to construct automations such as:

```text
DMR SMS
    |
    v
Authorization
    |
    v
Rule
    |
    v
Workflow
    |
    +----> Unlock door
    |
    +----> Send confirmation SMS
    |
    +----> Publish MQTT event
    |
    +----> Record audit entry
```

Or:

```text
Home Assistant
    |
    | MQTT
    v
DMR Gateway
    |
    +----> TTS announcement
    |
    +----> DMR SMS
    |
    +----> Cellular SMS
```

Or:

```text
GPIO sensor
    |
    v
Event
    |
    v
Rule
    |
    +----> DMR alert
    +----> MQTT
    +----> SMS
    +----> HTTP webhook
```

The central architectural idea is:

```text
EVENT
  |
  v
RULE
  |
  v
WORKFLOW
  |
  v
ACTION
  |
  v
JOB
  |
  v
EXECUTION
```

This model prevents individual integrations from becoming tightly coupled.

---

# 3. Problem Statement

DMR infrastructure often exists as isolated systems.

A typical environment may contain:

- a DMR repeater;
- an MMDVM hotspot;
- a Raspberry Pi;
- GPIO-controlled equipment;
- a cellular modem;
- Home Assistant;
- IP cameras;
- door controllers;
- MQTT devices;
- HTTP APIs;
- monitoring systems.

Each system may have its own API, protocol, configuration and logging.

This creates several problems:

1. Radio events cannot easily trigger modern automation.
2. Automation systems cannot easily communicate back to radio users.
3. Hardware actions become custom scripts.
4. Security policy is fragmented.
5. Logs are distributed across multiple services.
6. There is no unified event history.
7. There is no consistent authorization model.
8. Testing automation often requires real hardware.
9. Adding another RF backend can require application-wide changes.
10. User-written automation is difficult to isolate safely.

DMR Gateway addresses these problems with a normalized event and action architecture.

---

# 4. Design Principles

## 4.1 Event First

Everything important should be represented as an event.

Examples:

```text
dmr.sms.received
dmr.ptt.started
mqtt.message.received
gpio.state.changed
cellular.sms.received
schedule.triggered
alert.triggered
function.completed
```

---

## 4.2 Hardware Is an Adapter

The core application should not directly manipulate GPIO, serial devices, radio hardware, modems, cameras or proprietary protocols.

Instead:

```text
Core
 |
 +--> Adapter
       |
       +--> Device
```

This isolates hardware-specific behavior.

---

## 4.3 Actions Are Pluggable

Adding a new action should not require redesigning the rules engine.

An action should implement a common interface.

Examples:

```text
dmr.send_sms
dmr.broadcast_audio
mqtt.publish
gpio.set
gpio.pulse
serial.write
http.request
hikvision.unlock
sms.send
function.execute
delay
condition
```

---

## 4.4 Security Is a Platform Feature

Security must not be bolted on later.

The system must assume:

- network requests can be malicious;
- DMR commands can be forged or unauthorized;
- users can make mistakes;
- user-written code is untrusted;
- external APIs can fail;
- credentials can leak;
- hardware actions can have real-world consequences.

---

## 4.5 Observable by Default

Every important operation should be traceable.

The system should answer:

> What happened?

> Why did it happen?

> Which rule caused it?

> Which version of the rule ran?

> Who or what triggered it?

> What actions executed?

> What failed?

A correlation ID should connect all related records.

---

## 4.6 Simulation First

Every dangerous action should support dry-run or simulation wherever practical.

The operator should be able to test:

```text
event
 -> rule
 -> conditions
 -> actions
```

without actually unlocking a door, toggling a relay or transmitting RF.

---

## 4.7 Raspberry Pi Friendly

The default installation should be suitable for modest hardware.

Avoid requiring:

- Kubernetes;
- a large cluster;
- heavyweight observability infrastructure;
- multiple mandatory databases;
- excessive containers.

Optional components should remain optional.

---

# 6. Goals

## Primary Goals

- Multi-backend DMR support.
- Unified event model.
- Rule-based automation.
- Workflow execution.
- DMR SMS support.
- DMR voice/TTS alerts.
- MQTT integration.
- Home Assistant integration.
- GPIO and serial control.
- Cellular SMS bridging.
- Secure Edge Functions.
- Real-time web UI.
- Audit logging.
- Event history.
- Simulation.
- Event replay.
- CLI administration.
- Docker deployment.
- Raspberry Pi compatibility.

## Operational Goals

- Easy installation.
- Safe defaults.
- Clear diagnostics.
- Graceful failure.
- Idempotent actions where possible.
- Reliable queued execution.
- Strong authorization.
- No direct hardware access from user code.

---

# 7. Non-Goals

The project should not initially attempt to become:

- a replacement for professional radio infrastructure;
- a full DMR repeater implementation;
- a generic cloud platform;
- an unrestricted serverless execution platform;
- a full enterprise IAM product;
- a general-purpose PLC;
- a replacement for Home Assistant.

Instead, it should integrate with those systems.

---

# 8. Core Concepts

The platform consists of the following major concepts:

```text
User
Role
Permission

Device
Device Capability

Radio
Talkgroup

Event
Rule
Condition
Workflow
Action
Job
Execution

Alert
Alert Execution

Edge Function
Function Version

Integration
Secret

Audit Entry
```

The relationship is approximately:

```text
Device
  |
  +--> generates Events
  |
  +--> receives Actions

Events
  |
  +--> Rules
          |
          +--> Workflows
                  |
                  +--> Actions
                          |
                          +--> Jobs
                                  |
                                  +--> Executions
```

---

# 9. System Architecture

## 9.1 High-Level Architecture

```text
                           +-----------------------+
                           |      Vue 3 UI         |
                           | Naive UI / Pinia      |
                           +-----------+-----------+
                                       |
                                  REST / WS
                                       |
                           +-----------v-----------+
                           |      Go API/Core      |
                           |                       |
                           | Auth                  |
                           | Devices               |
                           | Rules                 |
                           | Workflows             |
                           | Alerts                |
                           | Events                |
                           | Audit                 |
                           +-----------+-----------+
                                       |
                          +------------+------------+
                          |                         |
                          v                         v
                  +---------------+         +---------------+
                  | Event Bus     |         | Task Queue    |
                  | Redis         |         | Asynq         |
                  +-------+-------+         +-------+-------+
                          |                         |
                          +------------+------------+
                                       |
                              +--------v--------+
                              | Action Engine   |
                              +--------+--------+
                                       |
              +------------------------+-----------------------+
              |            |              |          |         |
              v            v              v          v         v
           DMR           MQTT           GPIO       HTTP      SMS
          Adapter       Adapter        Agent      Adapter   Gateway
              |
       +------+------+ 
       |      |      |
     MMDVM  IPSC   Hytera
```

---

# 10. Trust Boundaries

The architecture deliberately separates components by trust level.

## Trust Zone 1: Core Application

Trusted application code:

```text
backend
worker
database
redis
```

## Trust Zone 2: Hardware

Privileged:

```text
hardware-agent
modem-gateway
DMR adapters
```

## Trust Zone 3: External Systems

Potentially untrusted or unreliable:

```text
HTTP APIs
MQTT clients
cellular networks
camera APIs
Home Assistant
```

## Trust Zone 4: User Code

Highly restricted:

```text
Deno runtime
```

User functions must never receive:

- Docker socket;
- host filesystem;
- unrestricted network;
- database credentials;
- Redis credentials;
- hardware devices;
- arbitrary process execution.

---

# 11. Event-Driven Architecture

The event system is the central architecture.

## 11.1 Event Flow

```text
SOURCE
  |
  v
Adapter
  |
  v
Normalization
  |
  v
Event Bus
  |
  +----> Event Store
  |
  +----> Rules Engine
  |
  +----> WebSocket
  |
  +----> MQTT
  |
  +----> Metrics
```

---

## 11.2 Event Lifecycle

1. Source generates event.
2. Adapter receives event.
3. Adapter validates source data.
4. Adapter normalizes event.
5. Event receives unique ID.
6. Event receives correlation ID.
7. Event is published.
8. Event is persisted.
9. Rules engine evaluates matching rules.
10. Matching rules produce workflow/action executions.
11. Jobs are queued.
12. Workers execute jobs.
13. Results are recorded.
14. UI receives live updates.
15. Audit record is created where applicable.

---

# 12. Event Model

Every event should have a common envelope.

```json
{
  "id": "evt_01J...",
  "type": "dmr.sms.received",
  "version": 1,
  "timestamp": "2026-10-01T01:10:00Z",
  "correlation_id": "corr_01J...",
  "source": {
    "type": "dmr",
    "backend": "mmdvm",
    "device_id": "device_01"
  },
  "subject": {
    "type": "radio",
    "id": "radio_01"
  },
  "payload": {},
  "metadata": {}
}
```

## 12.1 Event Fields

### id

Globally unique event ID.

### type

Namespaced event type.

Examples:

```text
dmr.sms.received
dmr.sms.sent
dmr.ptt.started
dmr.ptt.ended
dmr.voice.started
dmr.voice.ended
dmr.device.connected
dmr.device.disconnected
mqtt.message.received
gpio.changed
serial.received
cellular.sms.received
schedule.triggered
alert.triggered
workflow.started
workflow.completed
workflow.failed
function.started
function.completed
```

### version

Schema version.

### timestamp

UTC timestamp.

### correlation_id

Connects related events and actions.

### source

Identifies the origin.

### subject

Identifies the object affected.

### payload

Event-specific information.

### metadata

Optional adapter-specific or contextual information.

---

# 13. Device Abstraction

Devices are first-class resources.

```go
type Device struct {
    ID           uuid.UUID
    Name         string
    Type         string
    Driver       string
    Address      string
    Enabled      bool
    Config       json.RawMessage
    LastSeen     *time.Time
    CreatedAt    time.Time
    UpdatedAt    time.Time
}
```

Possible device types:

```text
mmdvm
ipsc
hytera
gpio
serial
modem
mqtt
camera
http
```

---

## 13.1 Device Capabilities

Devices advertise capabilities.

Examples:

```text
dmr.voice.receive
dmr.voice.transmit
dmr.sms.receive
dmr.sms.transmit
dmr.ptt.receive
dmr.status
gpio.read
gpio.write
serial.read
serial.write
sms.receive
sms.send
```

This allows the UI and action engine to determine what a device can actually do.

---

# 14. DMR Backend Architecture

All DMR implementations should expose a common adapter interface.

Conceptually:

```go
type DMRBackend interface {
    ID() string
    Connect(ctx context.Context) error
    Close() error
    Health(ctx context.Context) error

    Events() <-chan DMRRawEvent

    SendSMS(ctx context.Context, req SendSMSRequest) error
    BroadcastAudio(ctx context.Context, req BroadcastAudioRequest) error
}
```

The exact interface may evolve as backend capabilities are established.

The important requirement is that the core application must not depend on a specific DMR implementation.

---

# 15. MMDVM Integration

MMDVM is the primary initial RF backend.

The adapter should:

- receive radio activity;
- normalize PTT events;
- normalize voice lifecycle;
- receive DMR SMS where supported;
- transmit DMR SMS;
- transmit audio where supported;
- expose device health;
- expose backend diagnostics.

The MMDVM implementation should live behind the common DMR adapter.

Example:

```text
MMDVM
  |
  v
MMDVM Adapter
  |
  v
DMR Event Normalizer
  |
  v
Event Bus
```

The exact wire protocol and supported operations should be isolated inside the adapter.

---

# 16. Motorola IPSC Integration

Motorola IPSC support should be implemented as a separate adapter or bridge.

The bridge should translate proprietary/network-specific behavior into the normalized event model.

```text
Motorola Network
      |
      v
IPSC Bridge
      |
      v
Normalized DMR Events
      |
      v
Core Event Bus
```

The rest of the application must not need to know that an event originated from IPSC.

---

# 17. Hytera Integration

Hytera support should follow the same pattern.

```text
Hytera
   |
   v
Hytera Bridge
   |
   v
Normalized DMR Events
   |
   v
Core
```

The bridge should isolate:

- network protocol;
- authentication;
- proprietary packet formats;
- connection lifecycle;
- device-specific behavior.

---

# 18. Rules Engine

The rules engine converts events into automation.

A rule has:

```text
Trigger
Conditions
Actions / Workflow
Permissions
Rate Limits
Cooldown
Enabled State
Version
```

Example conceptual rule:

```yaml
name: Front Door Unlock

trigger:
  event: dmr.sms.received

conditions:
  - field: radio.talkgroup
    operator: equals
    value: 9001

  - field: radio.src_id
    operator: in
    value:
      - 1234567
      - 7654321

  - field: payload.text
    operator: matches
    value: "^UNLOCK [0-9]{4}$"

actions:
  - type: verify_pin
  - type: hikvision.unlock
    target: front-door
  - type: dmr.send_sms
    message: "Front door unlocked"

security:
  rate_limit:
    max: 3
    window: 60s
```

---

# 19. Conditions

Conditions should be composable.

Supported operators:

```text
equals
not_equals
contains
starts_with
ends_with
matches
in
not_in
greater_than
less_than
greater_or_equal
less_or_equal
exists
not_exists
```

Logical groups:

```text
AND
OR
NOT
```

Example:

```yaml
conditions:
  all:
    - field: radio.talkgroup
      equals: 9001

    - any:
        - field: radio.src_id
          equals: 1234567
        - field: radio.src_id
          equals: 7654321

    - field: payload.text
      matches: "^UNLOCK"
```

---

# 20. Actions

Actions are executable units.

Initial actions:

```text
dmr.send_sms
dmr.broadcast_audio
dmr.trigger_alert

mqtt.publish

gpio.set
gpio.pulse

serial.write

http.request

hikvision.unlock

sms.send

function.execute

delay
condition
sequence
parallel
```

Every action should support:

- validation;
- timeout;
- authorization;
- retry policy;
- dry-run where practical;
- structured result;
- audit information.

---

## 20.1 Action Interface

Conceptually:

```go
type Action interface {
    Type() string

    Validate(config map[string]any) error

    Execute(
        ctx context.Context,
        input ActionInput,
    ) (ActionResult, error)
}
```

---

# 21. Workflows

A workflow combines multiple actions.

Example:

```yaml
name: Doorbell Notification

steps:
  - id: mqtt
    action: mqtt.publish

  - id: radio
    action: dmr.broadcast_audio

  - id: wait
    action: delay
    config:
      duration: 10s

  - id: cellular
    action: sms.send
```

Workflows should support:

- sequential execution;
- parallel execution;
- conditional branches;
- delays;
- retries;
- timeouts;
- failure handling;
- compensation actions where appropriate;
- execution history.

---

## 21.1 Workflow Example

```text
Doorbell
   |
   +----> MQTT
   |
   +----> DMR announcement
   |
   +----> Wait 10 seconds
             |
             +----> If not acknowledged
                        |
                        +----> Cellular SMS
```

---

# 22. Task and Job System

Asynq should execute asynchronous work.

Examples:

```text
alert:trigger
tts:generate
dmr:audio_broadcast
dmr:sms_send
sms:cellular_send
mqtt:publish
action:gpio
action:serial
action:http
action:hikvision
function:execute
workflow:execute
```

Each job should have:

```text
job ID
type
payload
priority
attempt count
maximum attempts
created timestamp
started timestamp
completed timestamp
correlation ID
```

---

# 23. Alert System

Alerts are reusable definitions.

```go
type Alert struct {
    ID            uuid.UUID
    Name          string
    Type          string
    Text          string
    VoiceID       string
    TalkGroupID   uuid.UUID
    Enabled       bool
    LastTriggered *time.Time
    CreatedAt     time.Time
    UpdatedAt     time.Time
}
```

Alert types:

```text
tts
message
```

Potential future types:

```text
audio
workflow
multichannel
```

---

## 23.1 Alert Trigger

Endpoint:

```http
POST /api/alerts/{id}/trigger
```

The trigger should:

1. Authenticate request.
2. Authorize trigger permission.
3. Rate-limit request.
4. Validate alert state.
5. Create execution record.
6. Queue work.
7. Return execution ID.

The HTTP request should not wait for the entire radio transmission.

---

# 24. DMR Command System

Command talkgroups are dedicated channels for automation.

Example:

```text
9001 = DOORBELL
9002 = UNLOCK_FRONT
9003 = OPEN_GATE
9004 = SITE_ALARM
```

Commands can originate from:

```text
PTT
DMR SMS
```

A command must pass:

```text
Identity
Authorization
Rule matching
Rate limiting
Replay protection
Action authorization
```

---

# 25. Radio Identity and Authorization

Radio IDs should be represented as first-class entities.

Example:

```go
type Radio struct {
    ID          uuid.UUID
    DMRID       uint32
    Name        string
    Description string
    Enabled     bool
    CreatedAt   time.Time
    UpdatedAt   time.Time
}
```

Optional group membership:

```text
Security
Maintenance
Operators
Guests
Emergency
```

Permissions can then be assigned to groups.

---

## 25.1 Example

```text
Radio 1234567
    |
    v
Security Group
    |
    +----> unlock.front
    +----> gate.open
    +----> alarm.ack
```

A guest radio may have:

```text
dmr.receive
dmr.send
```

but not:

```text
door.unlock
gate.open
```

---

# 26. TTS Architecture

ElevenLabs is the initial TTS provider.

The TTS service should:

1. Accept text and voice configuration.
2. Calculate deterministic cache key.
3. Check cache.
4. Generate audio if missing.
5. Validate returned audio.
6. Store cached audio.
7. Return audio reference.
8. Queue DMR broadcast.

Cache key should include relevant synthesis parameters.

Example:

```text
SHA256(
    provider
    voice_id
    model
    text
    settings
)
```

This prevents duplicate API requests for identical audio.

---

# 27. MQTT and Home Assistant

MQTT is the primary automation integration.

Example event topics:

```text
dmr/events/sms_received
dmr/events/ptt_started
dmr/events/ptt_ended
dmr/events/voice_started
dmr/events/voice_ended
dmr/events/status
dmr/events/alert
```

Command topics:

```text
dmr/commands/alert
dmr/commands/send_sms
dmr/commands/broadcast
```

---

## 27.1 Home Assistant

Future support should include MQTT Discovery.

Potential entities:

```text
DMR Gateway online status
RF backend status
Last radio activity
Last talkgroup
Active alert
Connected modem
```

Buttons could trigger:

```text
Doorbell announcement
Site alarm
Maintenance announcement
```

---

# 28. Cellular SMS Gateway

Cellular SMS is an optional integration.

Architecture:

```text
DMR
 |
 v
Gateway
 |
 v
SMS Router
 |
 v
Modem / API Provider
 |
 v
Cellular Network
```

Inbound:

```text
Cellular SMS
 |
 v
Modem
 |
 v
Gateway
 |
 v
Normalized Event
 |
 v
Rule Engine
 |
 v
DMR SMS
```

Mapping should support:

```text
phone number -> talkgroup
phone number -> radio ID
phone number -> rule
```

---

# 29. Hardware Agent

The hardware agent is a privileged service.

It should provide controlled APIs for:

```text
GPIO
Serial
Modem
Other local devices
```

It must not expose arbitrary host command execution.

Example API:

```text
gpio.set
gpio.get
gpio.pulse

serial.list
serial.open
serial.write
serial.read
```

The backend should communicate with the hardware agent through authenticated local networking.

---

# 30. Edge Functions

Edge Functions provide user-defined automation.

Runtime:

```text
Deno
TypeScript
```

Functions should be short-lived and restricted.

Example:

```typescript
export default async function handler(event) {
    if (event.type === "dmr.sms.received") {
        await dmr.sendSms(
            event.radio.talkgroup,
            "Command received"
        );
    }
}
```

---

## 30.1 Function Security

Functions must have:

- CPU limit;
- memory limit;
- execution timeout;
- restricted filesystem;
- restricted network;
- no host device access;
- no Docker socket;
- no arbitrary process execution;
- explicit SDK permissions.

---

## 30.2 Function Permissions

Example:

```json
{
  "permissions": [
    "dmr.send_sms",
    "mqtt.publish"
  ]
}
```

A function lacking:

```text
hardware.gpio
```

must not be able to invoke GPIO.

---

# 31. Secrets Management

Sensitive credentials should be represented as secrets.

Examples:

```text
ElevenLabs API key
MQTT password
Camera password
Cellular provider credentials
DMR backend credentials
```

Secrets should:

- be encrypted at rest;
- never appear in normal API responses;
- never appear in logs;
- be masked in UI;
- be injected only into authorized processes;
- support rotation.

---

# 32. Authentication and Authorization

Initial authentication:

```text
Email
Password
JWT
```

Password storage:

```text
bcrypt
```

Potential future authentication:

```text
WebAuthn
Passkeys
OIDC
LDAP
```

---

## 32.1 Permissions

Instead of relying exclusively on roles, permissions should be granular.

Examples:

```text
alerts.read
alerts.write
alerts.trigger

talkgroups.read
talkgroups.write

devices.read
devices.write

rules.read
rules.write
rules.execute

workflows.read
workflows.write
workflows.execute

functions.read
functions.write
functions.execute

users.read
users.write

system.admin
audit.read
```

Roles can then be bundles of permissions.

---

# 33. Security Architecture

Security requirements:

- strong password hashing;
- secure JWT handling;
- short-lived access tokens;
- refresh token rotation where implemented;
- CSRF protection where applicable;
- rate limiting;
- input validation;
- audit logging;
- strict authorization;
- secure secrets storage;
- network segmentation;
- least privilege;
- sandboxed user code;
- no unnecessary privileged containers.

---

## 33.1 Sensitive Actions

Actions such as:

```text
door.unlock
gate.open
alarm.disable
```

should support additional policies:

```text
authorized radio
PIN
time window
cooldown
rate limit
confirmation
audit requirement
```

---

# 34. Audit Logging

Audit entries should contain:

```json
{
  "id": "audit_01",
  "timestamp": "2026-10-01T01:30:00Z",
  "actor_type": "radio",
  "actor_id": "1234567",
  "event": "door.unlock",
  "rule_id": "rule_01",
  "rule_version": 7,
  "action": "hikvision.unlock",
  "target": "front-door",
  "result": "success",
  "correlation_id": "corr_01"
}
```

Audit logs should be append-oriented and difficult for ordinary operators to modify.

---

# 35. Observability

The system should expose:

```text
Health
Metrics
Logs
Events
Executions
Audit
```

Prometheus-compatible metrics should be considered.

Examples:

```text
dmr_events_total
dmr_sms_total
dmr_voice_sessions_total
dmr_commands_total
dmr_command_failures_total

automation_executions_total
automation_execution_duration_seconds

tts_requests_total
tts_cache_hits_total

function_executions_total
function_execution_failures_total
```

---

# 36. Event Explorer

The UI should provide a real-time event explorer.

Example:

```text
01:32:14  MMDVM
          TG 9001 / TS1
          Radio 1234567
          PTT START

01:32:18  MMDVM
          TG 9001 / TS1
          VOICE END
          Duration: 4.2s

01:32:21  COMMAND
          "UNLOCK 4921"
          Rule: Front Door Unlock
          Authorization: PASS

01:32:21  HARDWARE
          GPIO 17 -> HIGH

01:32:22  DMR
          SMS -> 1234567
          "Front door unlocked"
```

Filters should include:

```text
time range
event type
device
backend
talkgroup
radio
rule
workflow
result
correlation ID
```

---

# 37. Simulation and Dry-Run

The system should include a simulator.

Example CLI:

```bash
dmrctl simulate sms \
  --src 1234567 \
  --tg 9001 \
  --text "UNLOCK 1234"
```

PTT:

```bash
dmrctl simulate ptt \
  --src 1234567 \
  --tg 9002
```

Dry-run should show:

```text
EVENT
  dmr.sms.received

MATCHED RULE
  Front Door Unlock v7

CONDITIONS
  PASS
  PASS
  PASS

ACTIONS

  [DRY RUN]
  hikvision.unlock(front-door)

  [DRY RUN]
  dmr.send_sms(...)
```

No physical action should occur.

---

# 38. Event Replay

Persisted events should be replayable.

Replay modes:

```text
inspect
dry-run
test-rule
test-workflow
```

Example:

```text
Event #183829
|
+-- Original Rule v4
|
+-- Replay using Rule v7
|
+-- Dry-run actions
```

This allows operators to test changes against historical events.

---

# 39. Web Application

Frontend:

```text
Vue 3
TypeScript
Vite
Naive UI
Pinia
VueUse
Monaco Editor
VeeValidate
Zod
```

Main views:

```text
Dashboard
Events
Devices
Radios
Talkgroups
Alerts
Rules
Workflows
Jobs
Functions
Integrations
Audit
Users
System
Settings
```

---

## 39.1 Dashboard

The dashboard should show:

```text
System Health
RF Backends
Recent Radio Activity
Active Jobs
Recent Automation
Alerts
Errors
MQTT Status
Modem Status
```

---

## 39.2 Rule Builder

Two modes should eventually exist.

### Visual Mode

Form-based rule creation.

### Advanced Mode

YAML/JSON editor.

The system should validate the rule before saving.

---

# 40. CLI

A CLI should complement the web UI.

Examples:

```bash
dmrctl status

dmrctl devices list

dmrctl radios list

dmrctl talkgroups list

dmrctl alerts list

dmrctl alerts trigger doorbell

dmrctl events tail

dmrctl events inspect <id>

dmrctl rules list

dmrctl rules test <id>

dmrctl workflows run <id>

dmrctl functions list

dmrctl functions invoke <id>

dmrctl audit tail

dmrctl simulate sms ...
```

---

# 41. Database Model

PostgreSQL is the primary database.

Core entities:

```text
users
roles
permissions
user_roles
role_permissions

devices
device_capabilities

radios
radio_groups
radio_group_members

talkgroups

alerts
alert_executions

rules
rule_versions
rule_executions

workflows
workflow_versions
workflow_executions

actions
jobs
job_executions

events
audit_logs

edge_functions
edge_function_versions

integrations
secrets

api_tokens
refresh_tokens
```

---

## 41.1 Event Storage

Events should be append-oriented.

Recommended fields:

```text
id
type
version
timestamp
correlation_id
source_type
source_id
subject_type
subject_id
payload
metadata
created_at
```

Indexes should exist for:

```text
timestamp
type
correlation_id
source_id
subject_id
```

---

# 42. API Design

Base path:

```text
/api
```

Authentication:

```text
POST /api/auth/login
POST /api/auth/logout
GET  /api/auth/me
```

Devices:

```text
GET    /api/devices
POST   /api/devices
GET    /api/devices/{id}
PUT    /api/devices/{id}
DELETE /api/devices/{id}
POST   /api/devices/{id}/test
```

Talkgroups:

```text
GET    /api/talkgroups
POST   /api/talkgroups
GET    /api/talkgroups/{id}
PUT    /api/talkgroups/{id}
DELETE /api/talkgroups/{id}
```

Alerts:

```text
GET    /api/alerts
POST   /api/alerts
GET    /api/alerts/{id}
PUT    /api/alerts/{id}
DELETE /api/alerts/{id}
POST   /api/alerts/{id}/trigger
```

Rules:

```text
GET    /api/rules
POST   /api/rules
GET    /api/rules/{id}
PUT    /api/rules/{id}
DELETE /api/rules/{id}
POST   /api/rules/{id}/test
GET    /api/rules/{id}/versions
```

Workflows:

```text
GET    /api/workflows
POST   /api/workflows
GET    /api/workflows/{id}
PUT    /api/workflows/{id}
DELETE /api/workflows/{id}
POST   /api/workflows/{id}/execute
POST   /api/workflows/{id}/dry-run
```

Events:

```text
GET /api/events
GET /api/events/{id}
POST /api/events/replay
```

Functions:

```text
GET    /api/functions
POST   /api/functions
GET    /api/functions/{id}
PUT    /api/functions/{id}
DELETE /api/functions/{id}
POST   /api/functions/{id}/execute
GET    /api/functions/{id}/versions
```

Audit:

```text
GET /api/audit
GET /api/audit/{id}
```

System:

```text
GET /api/system/health
GET /api/system/status
GET /api/system/metrics
```

---

# 43. WebSocket Design

Endpoint:

```text
/ws
```

The server should publish structured messages.

Example:

```json
{
  "type": "event.created",
  "data": {
    "event_id": "evt_01"
  }
}
```

Other messages:

```text
device.status
dmr.activity
alert.started
alert.completed
job.started
job.completed
workflow.started
workflow.completed
function.log
system.status
```

Clients should be able to subscribe to channels.

Example:

```json
{
  "action": "subscribe",
  "channels": [
    "dmr.activity",
    "jobs",
    "system"
  ]
}
```

---

# 44. Internal Service Contracts

Services should communicate using explicit contracts.

Preferred patterns:

```text
HTTP/gRPC for request/response
Redis event bus for events
Asynq for durable jobs
MQTT for external automation
```

Avoid direct database access from unrelated services.

For example:

```text
hardware-agent
```

should not query the application database directly.

---

# 45. Configuration

Configuration should support environment variables.

Example:

```env
APP_ENV=production
APP_LOG_LEVEL=info

HTTP_ADDR=:8080

DATABASE_URL=postgres://...
REDIS_URL=redis://...

JWT_SECRET=...

MQTT_URL=mqtt://mqtt:1883

ELEVENLABS_API_KEY=...

FUNCTION_RUNTIME_URL=http://deno-runtime:8090
HARDWARE_AGENT_URL=http://hardware-agent:8090
```

Sensitive values should eventually migrate to the secrets subsystem.

---

# 46. Docker Architecture

Initial mandatory services:

```text
frontend
backend
task-worker
postgres
redis
mqtt
```

Optional services:

```text
mmdvm
ipsc-bridge
hytera-bridge
hardware-agent
deno-runtime
modem-gateway
```

The default installation should not require all optional integrations.

---

# 47. Networking

Use separate Docker networks where useful.

Example:

```text
frontend-network
backend-network
integration-network
hardware-network
```

The database should not be publicly exposed.

Redis should not be publicly exposed.

MQTT authentication should be enabled for non-local deployments.

The hardware agent should only be reachable by authorized application services.

---

# 48. Persistence

Persistent data:

```text
PostgreSQL
Redis
TTS cache
Function source
MQTT configuration
Application configuration
```

TTS cache can use:

```text
Docker volume
```

or an object-storage abstraction in future versions.

---

# 49. Reliability

The system should prefer asynchronous execution.

A trigger endpoint should return:

```json
{
  "execution_id": "exec_01",
  "status": "queued"
}
```

rather than waiting for:

```text
TTS
RF transmission
SMS
camera action
```

to finish.

---

## 49.1 Idempotency

External triggers should support idempotency keys where practical.

Example:

```http
Idempotency-Key: 8c0c...
```

This prevents duplicate execution when clients retry requests.

---

# 50. Failure Handling

Actions should return structured results.

```json
{
  "status": "failed",
  "error_code": "DEVICE_OFFLINE",
  "message": "Front door controller unavailable"
}
```

Failures should be categorized:

```text
VALIDATION_ERROR
AUTHORIZATION_ERROR
DEVICE_OFFLINE
TIMEOUT
RATE_LIMITED
NETWORK_ERROR
PROVIDER_ERROR
INTERNAL_ERROR
```

---

# 51. Rate Limiting

Rate limits should exist at multiple levels.

## API

Per:

```text
IP
user
token
endpoint
```

## DMR

Per:

```text
radio
talkgroup
command
```

## Automation

Per:

```text
rule
action
target
```

Example:

```text
Unlock:
maximum 3 executions per minute
```

---

# 52. Validation

Validation must happen at boundaries.

Examples:

```text
API request
MQTT message
DMR message
Rule configuration
Workflow configuration
Function configuration
Device configuration
```

Use strict schemas.

Never assume external input is valid.

---

# 53. Testing Strategy

Testing should occur at several levels.

## Unit Tests

Test:

```text
rule evaluation
conditions
authorization
action validation
event normalization
configuration
```

## Integration Tests

Test:

```text
Postgres
Redis
MQTT
Asynq
HTTP APIs
```

## Adapter Tests

Use simulated RF inputs.

## End-to-End Tests

Example:

```text
Simulated DMR SMS
    |
    v
Event
    |
    v
Rule
    |
    v
Workflow
    |
    v
Simulated GPIO
    |
    v
Audit
```

---

# 54. Developer Experience

The repository should provide:

```bash
make dev
make test
make lint
make fmt
make migrate
make seed
make build
make docker
```

Local development should require as few external dependencies as possible.

A developer should be able to start the platform with:

```bash
docker compose up
```

and run the frontend/backend locally if desired.

---

# 55. Repository Structure

Recommended structure:

```text
dmr-gateway/
âââ README.md
âââ LICENSE
âââ CONTRIBUTING.md
âââ SECURITY.md
âââ CHANGELOG.md
âââ Makefile
âââ docker-compose.yml
âââ docker-compose.dev.yml
âââ .env.example
âââ .gitignore
â
âââ backend/
â   âââ cmd/
â   â   âââ api/
â   â   âââ worker/
â   â
â   âââ internal/
â   â   âââ domain/
â   â   â   âââ event/
â   â   â   âââ device/
â   â   â   âââ radio/
â   â   â   âââ talkgroup/
â   â   â   âââ alert/
â   â   â   âââ rule/
â   â   â   âââ workflow/
â   â   â   âââ action/
â   â   â   âââ function/
â   â   â
â   â   âââ application/
â   â   â   âââ events/
â   â   â   âââ automation/
â   â   â   âââ alerts/
â   â   â   âââ commands/
â   â   â
â   â   âââ adapters/
â   â   â   âââ dmr/
â   â   â   â   âââ mmdvm/
â   â   â   â   âââ ipsc/
â   â   â   â   âââ hytera/
â   â   â   âââ mqtt/
â   â   â   âââ gpio/
â   â   â   âââ serial/
â   â   â   âââ sms/
â   â   â   âââ http/
â   â   â
â   â   âââ infrastructure/
â   â   â   âââ postgres/
â   â   â   âââ redis/
â   â   â   âââ docker/
â   â   â   âââ logging/
â   â   â
â   â   âââ interfaces/
â   â       âââ http/
â   â       âââ websocket/
â   â
â   âââ migrations/
â   âââ go.mod
â   âââ go.sum
â   âââ Dockerfile
â
âââ frontend/
â   âââ src/
â   â   âââ components/
â   â   âââ layouts/
â   â   âââ views/
â   â   âââ stores/
â   â   âââ services/
â   â   âââ types/
â   â   âââ router/
â   âââ package.json
â   âââ Dockerfile
â
âââ hardware-agent/
â   âââ cmd/
â   âââ internal/
â   âââ Dockerfile
â   âââ go.mod
â
âââ hytera-bridge/
âââ ipsc-bridge/
âââ modem-gateway/
â
âââ deno-runtime/
â   âââ runtime/
â   âââ sdk/
â   âââ Dockerfile
â
âââ mosquitto/
â   âââ config/
â
âââ docs/
â   âââ architecture/
â   âââ api/
â   âââ events/
â   âââ rules/
â   âââ workflows/
â   âââ integrations/
â   âââ operations/
â
âââ scripts/
    âââ dev/
    âââ test/
    âââ release/
```

---

# 56. Technology Stack

## Backend

| Purpose | Technology |
|---|---|
| Language | Go |
| HTTP | chi |
| ORM | GORM |
| Database | PostgreSQL |
| Queue | Asynq |
| Cache/Event Bus | Redis |
| MQTT | Eclipse Paho |
| WebSocket | gorilla/websocket |
| Authentication | JWT |
| Password Hashing | bcrypt |
| Configuration | Viper |
| Validation | go-playground/validator |
| Logging | zerolog |
| Docker | Docker SDK |
| Serial | go.bug.st/serial |

## Frontend

| Purpose | Technology |
|---|---|
| Framework | Vue 3 |
| Language | TypeScript |
| Build | Vite |
| UI | Naive UI |
| State | Pinia |
| Utilities | VueUse |
| Editor | Monaco |
| Validation | Zod |
| Forms | VeeValidate |

## Infrastructure

```text
Docker
Docker Compose
PostgreSQL 16
Redis
Eclipse Mosquitto
Deno
```

---

# 57. Development Phases

## Phase 1 â Core Platform

Build:

- repository;
- Docker Compose;
- PostgreSQL;
- Redis;
- MQTT;
- Go API;
- authentication;
- users;
- devices;
- talkgroups;
- event model;
- WebSocket;
- basic Vue dashboard.

Deliverable:

```text
Platform boots
User can log in
Events can be created
Events appear live in UI
```

---

## Phase 2 â MMDVM

Build:

- MMDVM adapter;
- DMR event normalization;
- DMR SMS;
- outbound audio;
- device health.

Deliverable:

```text
Real DMR event
 -> Event Bus
 -> UI
```

---

## Phase 3 â Rules and Commands

Build:

- rule model;
- conditions;
- command authorization;
- action engine;
- job queue;
- audit logging.

Deliverable:

```text
DMR SMS
 -> rule
 -> simulated action
```

---

## Phase 4 â Hardware Agent

Build:

- GPIO;
- serial;
- device registration;
- health;
- safe action execution.

Deliverable:

```text
DMR command
 -> rule
 -> GPIO
```

---

## Phase 5 â MQTT / Home Assistant

Build:

- MQTT event publisher;
- MQTT command subscriber;
- discovery;
- integration management.

Deliverable:

```text
HA -> MQTT -> Gateway -> DMR
```

---

## Phase 6 â Alerts and TTS

Build:

- ElevenLabs integration;
- cache;
- alert execution;
- TTS broadcast.

Deliverable:

```text
HTTP
 -> Alert
 -> TTS
 -> DMR
```

---

## Phase 7 â Additional RF Backends

Build:

- Hytera bridge;
- Motorola IPSC bridge;
- capability discovery.

Deliverable:

```text
Multiple RF systems
       |
       v
Same event model
```

---

## Phase 8 â Cellular SMS

Build:

- modem adapter;
- SMS routing;
- inbound/outbound bridge;
- mapping rules.

---

## Phase 9 â Edge Functions

Build:

- Deno runtime;
- SDK;
- permission model;
- function versions;
- logs;
- sandboxing.

---

## Phase 10 â Production Hardening

Build:

- metrics;
- advanced audit;
- event replay;
- simulation;
- backup;
- recovery;
- security review;
- documentation;
- upgrade tooling.

---

# 58. Milestones

## M1 â Bootable

```text
docker compose up
```

works.

## M2 â Authenticated

Users can authenticate.

## M3 â Event System

Events can be created and streamed.

## M4 â First RF Backend

MMDVM events work.

## M5 â First Automation

A DMR message can trigger a simulated action.

## M6 â Real Hardware

GPIO action works through hardware-agent.

## M7 â Home Automation

MQTT integration works.

## M8 â TTS

Voice alert works.

## M9 â Multi-RF

Hytera/IPSC adapters work.

## M10 â Extensible Automation

Edge Functions work safely.

---

# 59. Operational Runbook

Operators should be able to answer:

### Is the gateway running?

```bash
docker compose ps
```

### Is the backend healthy?

```text
GET /api/system/health
```

### Is the RF backend connected?

Dashboard:

```text
Devices -> RF Backend -> Health
```

### Why did an automation fail?

Use:

```text
Events
 -> Correlation ID
 -> Rule Execution
 -> Workflow Execution
 -> Job
 -> Action
```

### Why didn't a rule trigger?

Use Rule Tester:

```text
event
 -> conditions
 -> authorization
 -> rate limit
```

---

# 60. Backup and Recovery

PostgreSQL is the primary source of configuration and historical data.

Backups should include:

```text
PostgreSQL
Function source
Configuration
Secrets metadata
MQTT configuration
```

TTS audio may be treated as rebuildable cache data.

Recommended backup types:

```text
daily database backup
weekly full backup
manual pre-upgrade backup
```

Restore should be tested periodically.

---

# 61. Upgrade Strategy

Version the application using semantic versioning.

```text
MAJOR.MINOR.PATCH
```

Database migrations must be forward-only and versioned.

Before upgrades:

```text
backup
stop risky automations
upgrade
migrate
health check
resume
```

The system should expose:

```text
application version
database migration version
frontend version
```

---

# 62. Performance Targets

These are initial engineering targets rather than hard guarantees.

## Event ingestion

Target:

```text
<100 ms
```

for local event normalization and publication under normal load.

## Rule evaluation

Target:

```text
<50 ms
```

for ordinary rule evaluation excluding external I/O.

## API

Target:

```text
p95 < 250 ms
```

for ordinary local CRUD requests.

## WebSocket

Events should appear in connected clients within approximately:

```text
<500 ms
```

under normal local operation.

External RF, TTS, MQTT, camera and cellular latency is outside the core application latency target.

---

# 63. Future Features

Potential future capabilities:

## Multi-Tenancy

Separate:

```text
organizations
sites
users
devices
rules
```

## Native Home Assistant Integration

A custom integration could expose:

```text
events
buttons
sensors
switches
diagnostics
```

## Mobile Application

Potential functions:

```text
alerts
events
acknowledgement
system health
remote administration
```

## Analytics

Examples:

```text
talkgroup activity
radio activity
alert frequency
automation failures
RF uptime
SMS volume
```

## Advanced Workflow Editor

A visual node-based editor could resemble:

```text
Trigger
   |
Condition
   |
 âââ´ââ
Yes No
 |   |
Action
 |
Parallel
 / \
A   B
 \ /
  C
```

---

# 64. Example Automations

## 64.1 Doorbell

```text
GPIO doorbell
    |
    v
gpio.changed
    |
    v
Rule
    |
    +----> DMR TTS
    |
    +----> MQTT
    |
    +----> Cellular SMS
```

---

## 64.2 Front Door

```text
DMR SMS
"UNLOCK 1234"
    |
    v
Check Talkgroup
    |
    v
Check Radio
    |
    v
Validate PIN
    |
    v
Rate Limit
    |
    v
Unlock
    |
    +----> Confirmation SMS
    |
    +----> MQTT
    |
    +----> Audit
```

---

## 64.3 Site Alarm

```text
Alarm input
    |
    v
GPIO event
    |
    v
Workflow
    |
    +----> DMR announcement
    |
    +----> DMR SMS
    |
    +----> Cellular SMS
    |
    +----> MQTT
    |
    +----> HTTP webhook
```

---

## 64.4 Scheduled Announcement

```text
08:00
 |
 v
Schedule Event
 |
 v
Alert
 |
 v
TTS
 |
 v
DMR Broadcast
```

---

## 64.5 Radio PTT Trigger

```text
PTT on TG 9002
    |
    v
Rule
    |
    +----> GPIO pulse
    |
    +----> MQTT event
    |
    +----> Confirmation
```

---

# 65. Example API Requests

## Trigger Alert

```http
POST /api/alerts/alert_123/trigger
Authorization: Bearer <token>
Idempotency-Key: 7f5c...
Content-Type: application/json
```

Response:

```json
{
  "execution_id": "exec_123",
  "status": "queued"
}
```

---

## Send DMR SMS

Conceptual internal request:

```json
{
  "talkgroup": 9001,
  "message": "Front door unlocked",
  "source": "automation"
}
```

---

## Create Rule

```json
{
  "name": "Front Door Unlock",
  "enabled": true,
  "trigger": {
    "event": "dmr.sms.received"
  },
  "conditions": {
    "all": [
      {
        "field": "radio.talkgroup",
        "operator": "equals",
        "value": 9001
      },
      {
        "field": "payload.text",
        "operator": "matches",
        "value": "^UNLOCK [0-9]{4}$"
      }
    ]
  }
}
```

---

# 66. Example Event Payloads

## DMR SMS

```json
{
  "id": "evt_01",
  "type": "dmr.sms.received",
  "version": 1,
  "timestamp": "2026-10-01T01:10:00Z",
  "correlation_id": "corr_01",
  "source": {
    "type": "dmr",
    "backend": "mmdvm",
    "device_id": "mmdvm_01"
  },
  "subject": {
    "type": "radio",
    "id": "radio_123"
  },
  "payload": {
    "src_id": 1234567,
    "dst_id": 9001,
    "talkgroup": 9001,
    "slot": 1,
    "text": "UNLOCK 1234"
  }
}
```

---

## PTT Start

```json
{
  "id": "evt_02",
  "type": "dmr.ptt.started",
  "version": 1,
  "timestamp": "2026-10-01T01:11:00Z",
  "correlation_id": "corr_02",
  "source": {
    "type": "dmr",
    "backend": "mmdvm",
    "device_id": "mmdvm_01"
  },
  "payload": {
    "src_id": 1234567,
    "talkgroup": 9002,
    "slot": 1
  }
}
```

---

## MQTT Event

```json
{
  "id": "evt_03",
  "type": "mqtt.message.received",
  "version": 1,
  "timestamp": "2026-10-01T01:12:00Z",
  "source": {
    "type": "mqtt",
    "device_id": "mqtt_01"
  },
  "payload": {
    "topic": "home/doorbell",
    "qos": 1,
    "retain": false,
    "body": {
      "pressed": true
    }
  }
}
```

---

# 67. Architectural Decisions

## ADR-001: Event-Driven Core

**Decision:** Use normalized events as the primary integration mechanism.

**Reason:** It decouples sources, rules and actions.

---

## ADR-002: DMR Adapters

**Decision:** RF protocols are adapters rather than core business logic.

**Reason:** MMDVM, IPSC and Hytera have different implementations and capabilities.

---

## ADR-003: Asynchronous Actions

**Decision:** Long-running operations use jobs.

**Reason:** RF transmission, TTS, SMS and external HTTP calls should not block API requests.

---

## ADR-004: Hardware Isolation

**Decision:** Physical hardware access occurs through hardware-agent.

**Reason:** Limits privilege and protects the core application.

---

## ADR-005: User Code Isolation

**Decision:** Edge Functions run in restricted Deno environments.

**Reason:** User code must be treated as untrusted.

---

## ADR-006: PostgreSQL as Source of Truth

**Decision:** Configuration, identities, rules, workflows and audit history live in PostgreSQL.

**Reason:** Relational integrity and transactional consistency are valuable for the control plane.

---

## ADR-007: Redis for Ephemeral Infrastructure

**Decision:** Redis is used for queues, transient state, pub/sub and caching.

**Reason:** Redis is well suited to fast ephemeral workloads.

---

## ADR-008: MQTT for External Automation

**Decision:** MQTT is the primary integration protocol for Home Assistant and similar systems.

**Reason:** It is lightweight, widely supported and appropriate for event-driven IoT automation.

---

# 68. Open Questions

The following should remain explicitly unresolved until implementation research/testing confirms the correct approach.

## RF

- Which MMDVM interfaces should be supported first?
- Which DMR SMS capabilities are available across each backend?
- Which audio formats are required by each backend?
- Which IPSC implementations are appropriate?
- Which Hytera bridge capabilities are reliable?

## Hardware

- Which Raspberry Pi GPIO library should become the default?
- Should hardware-agent use HTTP, gRPC or another local protocol?
- How should device permissions be represented?

## Security

- Should WebAuthn be included in the first major release?
- How should secrets be encrypted?
- How should key rotation work?
- Which sandboxing features are available in the target Deno/container environment?

## Database

- Should events remain in PostgreSQL indefinitely?
- Should old events be archived?
- Should event payloads be JSONB or partially normalized?

## Multi-Tenancy

- Is multi-tenancy required in the first major release?
- Should the data model be tenancy-ready from the beginning?

---

# 69. Definition of Done

A production-ready release should satisfy the following.

## Core

- [ ] Docker deployment works.
- [ ] Raspberry Pi ARM64 deployment works.
- [ ] PostgreSQL migrations work.
- [ ] Authentication works.
- [ ] RBAC works.
- [ ] API documentation exists.
- [ ] WebSocket updates work.

## Events

- [ ] Event envelope is versioned.
- [ ] Events have unique IDs.
- [ ] Events have correlation IDs.
- [ ] Events are persisted.
- [ ] Events are visible in UI.
- [ ] Event filtering works.

## DMR

- [ ] MMDVM adapter works.
- [ ] DMR SMS inbound works.
- [ ] DMR SMS outbound works.
- [ ] Voice activity events work.
- [ ] PTT events work.
- [ ] RF backend health is visible.

## Automation

- [ ] Rules work.
- [ ] Conditions work.
- [ ] Actions work.
- [ ] Workflows work.
- [ ] Jobs are durable.
- [ ] Retry policy works.
- [ ] Dry-run works.
- [ ] Rule testing works.

## Hardware

- [ ] GPIO agent works.
- [ ] Serial agent works.
- [ ] Hardware permissions work.
- [ ] Hardware actions are audited.

## Integrations

- [ ] MQTT works.
- [ ] Home Assistant integration works.
- [ ] TTS works.
- [ ] Cellular SMS works.
- [ ] Hytera adapter works.
- [ ] IPSC adapter works.

## Edge Functions

- [ ] Functions execute in isolation.
- [ ] Permissions work.
- [ ] Resource limits work.
- [ ] Function logs are streamed.
- [ ] Function versions are retained.

## Security

- [ ] Passwords are securely hashed.
- [ ] JWT handling is secure.
- [ ] Sensitive endpoints are rate limited.
- [ ] Audit logging works.
- [ ] Secrets are protected.
- [ ] User code cannot access host resources.
- [ ] No privileged container is unnecessarily exposed.

## Operations

- [ ] Health endpoints exist.
- [ ] Metrics exist.
- [ ] Backup procedure exists.
- [ ] Restore procedure has been tested.
- [ ] Upgrade procedure is documented.
- [ ] Failure modes are documented.

---

# 70. Conclusion

DMR Gateway should not be implemented as a collection of independent scripts for DMR, GPIO, MQTT, SMS and HTTP.

The long-term architecture should instead be centered around a common automation model:

```text
                    +----------------+
                    |     SOURCES    |
                    +--------+-------+
                             |
                             v
                    +----------------+
                    | NORMALIZED     |
                    | EVENTS         |
                    +--------+-------+
                             |
                             v
                    +----------------+
                    | RULE ENGINE    |
                    +--------+-------+
                             |
                             v
                    +----------------+
                    | WORKFLOWS      |
                    +--------+-------+
                             |
                             v
                    +----------------+
                    | ACTION ENGINE  |
                    +--------+-------+
                             |
                             v
                    +----------------+
                    | JOB EXECUTION  |
                    +--------+-------+
                             |
          +------------------+------------------+
          |                  |                  |
          v                  v                  v
        DMR               Hardware          External
        RF                GPIO/Serial       Services
```

This architecture provides the foundation for a system that can start small on a Raspberry Pi and grow into a sophisticated radio automation platform.

The most important engineering decisions are:

1. **Keep RF protocols behind adapters.**
2. **Make events first-class objects.**
3. **Separate rules from actions.**
4. **Separate actions from jobs.**
5. **Treat workflows as composable automation.**
6. **Keep hardware behind a privileged agent.**
7. **Treat user code as untrusted.**
8. **Make authorization explicit for sensitive actions.**
9. **Make auditability and observability first-class.**
10. **Build simulation and replay before the platform becomes difficult to test.**

The resulting system is not merely a DMR controller.

It is an extensible, self-hosted automation platform where DMR becomes a first-class participant in a broader event-driven ecosystem.

---

## Suggested Initial Implementation Order

```text
1. Repository + Docker Compose
2. PostgreSQL + migrations
3. Go API
4. Authentication
5. Event model
6. Redis event bus
7. WebSocket hub
8. Vue dashboard
9. Device model
10. Talkgroup model
11. MMDVM adapter
12. Rule engine
13. Action engine
14. Asynq workers
15. Simulation framework
16. GPIO hardware-agent
17. MQTT
18. TTS
19. Alerts
20. Audit/event explorer
21. Hytera/IPSC
22. Cellular SMS
23. Edge Functions
24. Replay and advanced observability
```

This order deliberately builds the reusable platform primitives before adding the more specialized integrations.
