# Agent Registry and Discovery

<!-- SPDX-License-Identifier: MIT -->

## Overview

The **Agent Registry** is a distributed capability for agents to discover each other, validate skill profiles, and coordinate on resource availability.

This document describes the registry patterns and MQTT topics used for agent discovery.

## Registry Patterns

### Service Announcement

Agents announce their availability on system startup:

```
Topic: $a2a/v1/registry/announce/{org}/{circle}/{agentId}
Payload: {
  "agentId": "e1f66962-dc3c-4a8e-9875-de1a1dee2839",
  "circle": "engineering",
  "roles": ["circle-lead", "backend-engineer"],
  "skills": ["rate-limiter", "distributed-systems", "api-design"],
  "status": "online",
  "timestamp": "2026-06-05T08:30:00Z"
}
```

Retention: Yes (agent continues broadcasting every 30-60 seconds)

### Heartbeat/Keep-Alive

Agents periodically send heartbeats to signal continued availability:

```
Topic: $a2a/v1/registry/heartbeat/{org}/{circle}/{agentId}
Payload: {
  "agentId": "e1f66962-dc3c-4a8e-9875-de1a1dee2839",
  "timestamp": "2026-06-05T08:35:00Z",
  "capacity": { "pending_tasks": 3, "max_concurrent": 10 }
}
```

Frequency: Every 30-60 seconds

### Agent Inquiry

Query the registry for agents with a specific skill:

```
Topic: $a2a/v1/registry/query/{org}
Request: {
  "queryId": "q-123",
  "kind": "find-agents-by-skill",
  "skill": "security-audit",
  "circle": null,  // null = cross-circle
  "responseId": "...",
  "requestorId": "..."
}
```

### Registry Response

Registry service responds with matching agents:

```
Topic: $a2a/v1/registry/response/{org}/query-123
Payload: {
  "queryId": "query-123",
  "matches": [
    {
      "agentId": "audit-agent-1",
      "circle": "compliance",
      "skills": ["security-audit", "risk-analysis"],
      "status": "online",
      "trust_scores": { "security-audit": 0.92 }
    },
    {
      "agentId": "audit-agent-2",
      "circle": "compliance",
      "skills": ["security-audit"],
      "status": "online",
      "trust_scores": { "security-audit": 0.75 }
    }
  ]
}
```

## Trust Scoring

Agents maintain trust scores on each skill:

| Trust Level | Score | Meaning |
|-------------|-------|---------|
| Stranger | 0.0-0.5 | New agent, not yet vetted |
| Endorsed | 0.5-0.7 | Endorsed by senior peer |
| Trusted | 0.7-0.85 | Proven track record |
| Running-Mate | 0.85+ | Highly trusted, autonomous |

Registry may return trust scores alongside agent profiles for skill-based routing decisions.

## Discovery Workflow

### Step 1: Agent Comes Online

1. Agent generates its ID (UUID)
2. Agent connects to MQTT broker
3. Agent subscribes to `$a2a/v1/registry/announce/{org}/{circle}/#`
4. Agent publishes announcement on its own registry topic
5. Agent starts heartbeat loop

### Step 2: Other Agents Detect New Agent

1. Agents subscribed to circle's registry see announcement
2. May update local caches of available agents
3. Do NOT need to query registry for each task

### Step 3: Task Sender Queries for Skill Match

1. Task sender publishes query to `$a2a/v1/registry/query/{org}`
2. Central registry service (or distributed peers) match and respond
3. Task sender picks best-fit agent by trust score
4. Task sender routes task to chosen agent

## Implementation Notes

- Agents do **not** require a central registry server
- Agents can use local MQTT topic subscriptions to maintain agent lists
- Registry is eventually consistent (agents may take 30-60 seconds to appear)
- Trust scores can be computed offline from task history

## Optional: Centralized Registry

Organizations may deploy a registry microservice that:

- Subscribes to all `$a2a/v1/registry/announce` topics
- Maintains an in-memory agent directory
- Responds to `$a2a/v1/registry/query` requests
- Provides REST API for third-party tools

This is optional and not required by the A2A protocol itself.

