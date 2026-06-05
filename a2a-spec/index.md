---
layout: default
title: A2A-over-MQTT Protocol Specification
permalink: /a2a-spec/
redirect_from:
  - /spec/
  - /a2a-spec/index.html
---

# A2A-over-MQTT Protocol Specification v1.0.0

<!-- SPDX-License-Identifier: MIT -->
<!-- Copyright (c) 2026 ContextHero -->
<!-- License: MIT (see LICENSE file) -->

**Status**: Released  
**Version**: 1.0.0  
**Release Date**: 2026-06-05  
**License**: MIT

---

## 1. Introduction

The **A2A-over-MQTT Protocol** defines a lightweight, distributed pattern for autonomous software agents to communicate peer-to-peer over MQTT v5. It enables:

- **Directed task requests** with automatic reply correlation
- **Broadcast events** scoped to organizational circles or skill-based groups
- **Asynchronous coordination** without centralized routing
- **Distributed authority** via role-based message routing
- **Language-agnostic** implementation (any MQTT v5 client)

This specification is published under the **MIT License** to maximize ecosystem adoption and enable derivative use.

---

## 2. Protocol Architecture

### 2.1 Core Concepts

**Agent**: An autonomous process holding roles in an organizational structure. Each agent has a unique UUID and skill profile.

**Circle**: A bounded namespace for authority and accountability (from Holacracy). Agents are grouped into circles by organizational role.

**Organization (Org)**: The top-level namespace containing one or more circles.

**Topic**: An MQTT topic path routing messages to agents, circles, or skill-based groups.

**Correlation**: MQTT v5 native feature (Response-Topic + Correlation-Data) enabling request-reply semantics.

### 2.2 Communication Patterns

| Pattern | Use Case | QoS | Durability |
|---------|----------|-----|-----------|
| **Directed Task** | One agent → one agent with reply | 1 | Until reply correlates |
| **Role Broadcast** | One agent → all agents in a role | 0 | Fire-and-forget |
| **Circle Broadcast** | One agent → all in circle | 0 | Fire-and-forget |
| **Skill Broadcast** | One agent → all holding a skill | 0 | Fire-and-forget |
| **Pool Ask** | One agent → one filler of role (round-robin) | 1 | Until one replies |

---

## 3. Topic Architecture

### 3.1 Topic Syntax

All A2A topics follow the pattern:

```
$a2a/v1/{messageType}/{org}/{unit}/{targetId}
```

**Components:**
- `$a2a` — Protocol namespace (prevents collision with app topics)
- `v1` — Protocol version
- `{messageType}` — Type of message (request, reply, event, etc.)
- `{org}` — Organization UUID or slug
- `{unit}` — Circle/group UUID (or "self" for broadcasts)
- `{targetId}` — Target agent UUID or "self"

### 3.2 Topic Types

#### Request Topics (Directed Task)

```
$a2a/v1/request/{org}/{circle}/{toAgentId}
```

**Used for**: Sending a directed task to a specific agent.

**Properties**:
- Sender sets MQTT v5 `Response-Topic` and `Correlation-Data`
- Receiver subscribes to this topic with agent-specific wildcard
- Must not be retained

#### Reply Topics (Directed Response)

```
$a2a/v1/reply/{org}/{circle}/{toAgentId}/{taskId}
```

**Used for**: Returning results to the original requester.

**Properties**:
- Populated from the original request's Response-Topic
- Correlation-Data echoed in the reply message
- Must not be retained
- Should use QoS 1 to ensure delivery

#### Event Topics (Broadcast)

```
$a2a/v1/event/{org}/{circle}/{fromAgentId}
```

**Used for**: Broadcasting events to all subscribers in a circle.

**Properties**:
- Fire-and-forget (QoS 0)
- Subscribers use wildcard: `$a2a/v1/event/{org}/{circle}/#`
- Not retained

#### Role Broadcast Topics

```
$a2a/v1/role-broadcast/{org}/{circle}/{roleId}/{fromAgentId}
```

**Used for**: Broadcasting to all agents filling a specific role.

**Properties**:
- Fire-and-forget (QoS 0)
- All role fillers subscribe to: `$a2a/v1/role-broadcast/{org}/{circle}/{roleId}/#`

#### Skill Broadcast Topics

```
$a2a/v1/skill-broadcast/{org}/{skill}/{fromAgentId}
```

**Used for**: Broadcasting to all agents holding a skill (cross-circle).

**Properties**:
- Cross-organizational scope
- All skill-holders subscribe to: `$a2a/v1/skill-broadcast/{org}/{skill}/#`

---

## 4. Message Format

### 4.1 Directed Task Request

**Topic**: `$a2a/v1/request/{org}/{circle}/{toAgentId}`

**MQTT v5 Properties**:
```json
{
  "Response-Topic": "$a2a/v1/reply/{org}/{circle}/{fromAgentId}/{taskId}",
  "Correlation-Data": "{taskId}",
  "User-Properties": {
    "from-agent-id": "{fromAgentId}",
    "message-kind": "task",
    "timeout-ms": "60000"
  }
}
```

**Payload** (JSON):
```json
{
  "taskId": "550e8400-e29b-41d4-a716-446655440000",
  "kind": "task",
  "title": "Review pull request",
  "description": "Please review PR #42 for security issues",
  "requiredSkills": ["security-review"],
  "timeoutMs": 60000,
  "createdAt": "2026-06-05T08:30:00Z"
}
```

### 4.2 Directed Task Reply

**Topic**: Set from original request's Response-Topic  
**Correlation-Data**: Echoed from original request

**Payload** (JSON):
```json
{
  "taskId": "550e8400-e29b-41d4-a716-446655440000",
  "status": "completed",
  "result": {
    "approved": true,
    "findings": ["Config uses deprecated cipher suite"]
  },
  "completedAt": "2026-06-05T08:35:00Z"
}
```

### 4.3 Broadcast Event

**Topic**: `$a2a/v1/event/{org}/{circle}/{fromAgentId}`

**Payload** (JSON):
```json
{
  "kind": "permission-granted",
  "body": {
    "agentId": "2b9fda1c-5163-4ab9-9a52-e955e95c93ac",
    "resource": "github-org:ContextHero",
    "permission": "repo-creator"
  },
  "createdAt": "2026-06-05T08:30:00Z"
}
```

---

## 5. Subscription Patterns

### 5.1 Agent Subscribes to Its Requests

```
$a2a/v1/request/{org}/{circle}/{agentId}
```

Every agent subscribes to its own agent-specific request topic.

### 5.2 Agent Subscribes to Its Replies

```
$a2a/v1/reply/{org}/{circle}/{agentId}/#
```

Agent uses wildcard to receive all replies targeted to it.

### 5.3 Agent Subscribes to Circle Events

```
$a2a/v1/event/{org}/{circle}/#
```

All agents in a circle can subscribe to see events from any agent in the circle.

### 5.4 Agent Subscribes to Role Broadcasts

```
$a2a/v1/role-broadcast/{org}/{circle}/{roleId}/#
```

Only agents filling a specific role subscribe to role-specific broadcasts.

### 5.5 Agent Subscribes to Skill Broadcasts

```
$a2a/v1/skill-broadcast/{org}/{skill}/#
```

Cross-circle: all agents with a skill subscribe to its broadcast channel.

---

## 6. Correlation and Reply Routing

### 6.1 Request Correlation

When sending a directed task:

1. Sender generates a unique `taskId` (UUID v4 or similar)
2. Sender sets MQTT v5 `Response-Topic` to:
   ```
   $a2a/v1/reply/{org}/{circle}/{fromAgentId}/{taskId}
   ```
3. Sender sets `Correlation-Data` to the taskId (binary or string)
4. Sender publishes request on target's topic with QoS 1
5. Receiver processes the task and publishes reply to the Response-Topic
6. Sender correlates reply by matching taskId in the reply topic or Correlation-Data

### 6.2 Timeout Handling

- Requesters should implement a timeout (recommended: 60 seconds for default tasks)
- If no reply arrives before timeout, the task is considered failed
- Requesters **must not** permanently subscribe to reply topics — unsubscribe after timeout or receipt

### 6.3 Idempotence

Receivers should track processed taskIds to avoid duplicate processing in case of retransmission.

---

## 7. Error Handling

### 7.1 Task Rejection

If a receiver cannot process a task, it should publish a reply with `status: "declined"`:

```json
{
  "taskId": "...",
  "status": "declined",
  "reason": "skill-trust-below-threshold",
  "details": "Agent lacks required skill 'security-review' with sufficient trust"
}
```

### 7.2 Malformed Messages

Receivers **must** ignore messages with:
- Missing required fields
- Invalid JSON
- Unrecognized message kinds

### 7.3 Connection Loss

- Agents should reconnect and re-subscribe to their topics
- Pending replies are lost (requesters timeout and retry)
- Broadcasts (QoS 0) are not delivered to offline agents
- Directed tasks (QoS 1) are retained by broker until delivered

---

## 8. Security Considerations

### 8.1 Authentication

- All A2A communication occurs over an authenticated MQTT connection
- Agent identity is derived from MQTT client credentials (username/certificate)
- Agents must validate that incoming messages are from expected peers

### 8.2 Authorization

- Organizations may implement access control lists (ACLs) to restrict which agents can publish to certain topics
- Brokers should enforce topic-based ACLs per agent credentials

### 8.3 Encryption

- All MQTT connections should use TLS
- Payload encryption is optional (delegated to application layer)

### 8.4 Rate Limiting

- Brokers may implement rate limits per agent to prevent DoS
- High-volume broadcasts should be rate-limited or filtered

---

## 9. Examples

### 9.1 Directed Task with Reply

**Agent A sends task to Agent B:**

```
Topic: $a2a/v1/request/myorg/engineering/b-uuid
Response-Topic: $a2a/v1/reply/myorg/engineering/a-uuid/task-123
Correlation-Data: task-123
Payload:
{
  "taskId": "task-123",
  "kind": "approve-pr",
  "title": "Review MR #42",
  "requiredSkills": ["code-review"],
  "timeoutMs": 60000
}
```

**Agent B receives, processes, and replies:**

```
Topic: $a2a/v1/reply/myorg/engineering/a-uuid/task-123
Correlation-Data: task-123
Payload:
{
  "taskId": "task-123",
  "status": "completed",
  "result": { "approved": true }
}
```

### 9.2 Circle Broadcast Event

**Agent A publishes an event to its circle:**

```
Topic: $a2a/v1/event/myorg/product/a-uuid
Payload:
{
  "kind": "release-published",
  "body": {
    "version": "1.0.0",
    "url": "https://github.com/myorg/repo/releases/tag/v1.0.0"
  }
}
```

**All agents in the circle (subscribed to `$a2a/v1/event/myorg/product/#`) receive the event.**

### 9.3 Skill-Based Round-Robin Ask

**Coordinator asks a pool of auditors:**

```
Topic: $a2a/v1/request-pool/myorg/audit/coordinator-uuid
Response-Topic: $a2a/v1/reply/myorg/audit/coordinator-uuid/audit-req-456
Correlation-Data: audit-req-456
Payload:
{
  "taskId": "audit-req-456",
  "kind": "financial-audit",
  "poolRequiredSkill": "financial-auditor",
  "title": "Q2 2026 Financial Audit"
}
```

**Broker round-robins among agents with the `financial-auditor` skill. The first to subscribe (or available) processes and replies.**

---

## 10. Implementation Notes

### 10.1 Client Libraries

Implementers should provide SDKs in:
- Python (asyncio)
- JavaScript/Node.js
- Go
- Rust

### 10.2 Broker Requirements

- MQTT v5 support (Response-Topic, Correlation-Data)
- QoS 0 and 1 support
- Wildcard subscriptions ($, #)
- User properties support

### 10.3 Backwards Compatibility

This is version 1.0.0. Future versions will maintain backwards compatibility within the v1 lineage. Breaking changes require a new major version.

---

## 11. License

This specification is published under the **MIT License**. See [LICENSE](../LICENSE) for full text.

You are free to:
- Use this specification to build A2A implementations
- Modify and extend this specification
- Distribute derivative specifications or implementations

With the requirement that you include the original license.

---

## 12. References

- [MQTT v5.0 Specification](https://docs.oasis-open.org/mqtt/mqtt/v5.0/os/mqtt-v5.0-os.html)
- [Holacracy Constitution](https://www.holacracy.org/)
- [SPDX License List (MIT)](https://spdx.org/licenses/MIT.html)

---

**Document Status**: Final Release  
**Last Updated**: 2026-06-05  
**Maintainer**: ContextHero Engineering Circle

---

## Companion documents

- [MQTT Topic Architecture](mqtt-topics.html)
- [Agent Registry](agent-registry.html)
- [Examples directory](https://github.com/ContextHero/a2a-mqtt-spec/tree/main/examples)
