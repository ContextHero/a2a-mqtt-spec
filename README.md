# A2A-over-MQTT Protocol Specification

[![MIT License](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

A distributed, agent-to-agent (A2A) protocol for asynchronous message-oriented communication over MQTT, enabling autonomous software agents to coordinate work across organizational boundaries without centralized routing.

## Overview

The **A2A-over-MQTT Protocol** is a lightweight specification for peer-to-peer agent communication built on MQTT v5. It defines how autonomous agents discover each other, send directed tasks, broadcast events, and maintain conversation state across disconnects.

### Key Features

- **Peer-to-peer messaging**: Direct agent-to-agent requests with automatic reply routing
- **Broadcast primitives**: Fire-and-forget events scoped to circles or skills
- **MQTT v5 native**: Uses response-topic + correlation-data for reply correlation
- **Holacracy-aware**: Designed to support distributed authority via role-based routing
- **Language-agnostic**: Works with any MQTT client library

## Quick Start

See the [specification document](./docs/a2a-mqtt-spec.md) for full protocol details.

### Example: Sending a Directed Task

An agent can send a task to a peer and await a reply:

```
Topic: $a2a/v1/request/{org}/{circle}/{toAgentId}
Response-Topic: $a2a/v1/reply/{org}/{circle}/{fromAgentId}/{taskId}
Correlation-Data: {taskId}

Message payload: JSON-encoded task
```

The receiver processes the task and publishes a reply on the response-topic.

## Use Cases

- **Distributed governance**: Agents route work via role accountability, not hierarchy
- **Ecosystem coordination**: Cross-org agent networks without API gateways
- **Autonomous workflows**: Agents delegate work peer-to-peer, coordinating skill fit and capacity
- **Event broadcasting**: Publish organization-wide events with subscriber filtering

## Specification Documents

- [A2A-over-MQTT Protocol Specification](./docs/a2a-mqtt-spec.md) — Full protocol definition
- [MQTT Topic Architecture](./docs/mqtt-topics.md) — Topic naming and routing rules
- [Agent Registry](./docs/agent-registry.md) — Discovery and registration patterns

## License

This specification is released under the **MIT License**. See [LICENSE](LICENSE) for details.

## Contributing

Contributions are welcome. Please open an issue or PR to discuss proposed changes.

---

**Status**: Released 2026-06-05  
**Version**: 1.0.0
