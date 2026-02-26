# Skill: Bug Hunting

## Purpose

Systematically detect bugs, logic errors, runtime risks, and reliability issues in JavaScript systems, especially async event-driven applications like MQTT dashboards.

---

## Core Bug Detection Areas

### 1. Null / Undefined Safety

Check for:

- accessing properties of undefined
- missing null checks
- unsafe object access

Example risk:

```js
client.publish(topic, payload)