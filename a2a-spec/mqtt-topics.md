---
layout: default
title: MQTT Topic Architecture
permalink: /a2a-spec/mqtt-topics.html
---

# A2A-over-MQTT Topic Architecture

<!-- SPDX-License-Identifier: MIT -->

## Topic Naming Convention

All A2A topics follow the structure:

```
$a2a/v1/{messageType}/{org}/{unit}/{targetId}
```

### Components

| Component | Description | Example |
|-----------|-------------|---------|
| `messageType` | Type of message (request, reply, event, broadcast) | `request`, `reply`, `event` |
| `org` | Organization UUID or slug | `myorg`, `46cad2c0-19f3-4a22-95d1-c5f3dcb0f096` |
| `unit` | Circle UUID or `self` for broadcasts | `engineering`, `self` |
| `targetId` | Target agent UUID | `e1f66962-dc3c-4a8e-9875-de1a1dee2839` |

## Topic Types Reference

### Directed Communication

- **Request**: `$a2a/v1/request/{org}/{circle}/{toAgentId}`
  - Used for: Sending a directed task to a specific agent
  - QoS: 1 (ensure delivery)
  - Retained: No

- **Reply**: `$a2a/v1/reply/{org}/{circle}/{fromAgentId}/{taskId}`
  - Used for: Returning task results
  - QoS: 1 (ensure delivery)
  - Retained: No

### Broadcasting

- **Circle Event**: `$a2a/v1/event/{org}/{circle}/{fromAgentId}`
  - Used for: Broadcasting to all agents in a circle
  - Subscribe: `$a2a/v1/event/{org}/{circle}/#`
  - QoS: 0 (fire-and-forget)

- **Role Broadcast**: `$a2a/v1/role-broadcast/{org}/{circle}/{roleId}/{fromAgentId}`
  - Used for: Broadcasting to all agents filling a role
  - Subscribe: `$a2a/v1/role-broadcast/{org}/{circle}/{roleId}/#`
  - QoS: 0 (fire-and-forget)

- **Skill Broadcast**: `$a2a/v1/skill-broadcast/{org}/{skill}/{fromAgentId}`
  - Used for: Broadcasting to all agents with a skill (cross-circle)
  - Subscribe: `$a2a/v1/skill-broadcast/{org}/{skill}/#`
  - QoS: 0 (fire-and-forget)

### Pooled Requests

- **Role Pool Ask**: `$a2a/v1/request-pool/{org}/{circle}/{roleId}`
  - Used for: Round-robin dispatch to one agent filling a role
  - QoS: 1 (ensure delivery)
  - Broker behavior: Routes to next available filler (round-robin)

- **Skill Pool Ask**: `$a2a/v1/request-pool/{org}/{skill}`
  - Used for: Round-robin dispatch to one agent with a skill (cross-circle)
  - QoS: 1 (ensure delivery)

## Subscription Strategies

### For Agents

An agent should subscribe to:

1. **Its own request topic**:
   ```
   $a2a/v1/request/{org}/{circle}/{agentId}
   ```

2. **Its reply topics** (to receive task results):
   ```
   $a2a/v1/reply/{org}/{circle}/{agentId}/#
   ```

3. **Circle events** (if in a circle):
   ```
   $a2a/v1/event/{org}/{circle}/#
   ```

4. **Role broadcasts** (if filling a role):
   ```
   $a2a/v1/role-broadcast/{org}/{circle}/{roleId}/#
   ```

5. **Skill broadcasts** (optional, if holding a skill):
   ```
   $a2a/v1/skill-broadcast/{org}/{skillSlug}/#
   ```

### For External Monitors/Audits

Monitoring systems may subscribe to:

```
$a2a/v1/event/{org}/{circle}/#
```

to see all events in a circle without processing directed tasks.

## Examples

### Example 1: Task Request and Reply

Request from `dev-lead` to `backend-eng` in `engineering` circle:

```
Publish to: $a2a/v1/request/myorg/engineering/backend-eng-uuid
Response-Topic: $a2a/v1/reply/myorg/engineering/dev-lead-uuid/task-001
Correlation-Data: task-001
```

Response from `backend-eng`:

```
Publish to: $a2a/v1/reply/myorg/engineering/dev-lead-uuid/task-001
Correlation-Data: task-001
```

### Example 2: Circle Broadcast

Event from `dev-lead`:

```
Publish to: $a2a/v1/event/myorg/engineering/dev-lead-uuid
Subscribers: $a2a/v1/event/myorg/engineering/#
```

### Example 3: Skill Broadcast

Announcement from any agent to all security auditors:

```
Publish to: $a2a/v1/skill-broadcast/myorg/security-audit/sender-uuid
Subscribers: All agents with skill 'security-audit' subscribe to: $a2a/v1/skill-broadcast/myorg/security-audit/#
```

## Reserved Prefixes

- `$` — MQTT system topics (SYS topics)
- `$a2a/` — Reserved for A2A protocol
- Custom prefixes should not use `$` to avoid conflicts

