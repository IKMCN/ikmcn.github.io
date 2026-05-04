---
layout: post
title: "The single MQTT code path"
description: "Why all state changes — from hardware, API, or simulator — funnel through one broker topic structure."
date: 2026-03-01
category: Architecture
next_title: "Schema design for multi-tenancy"
next_url: /schema-design-for-multi-tenancy/
---

When I started building the launderette IoT platform, I had three different sources that could change a machine's state: the physical hardware client on the machine, direct API calls from the admin frontend, and a WPF simulator I was building to test the stack. The obvious temptation is to handle each one differently — the hardware publishes to MQTT, the API writes directly to the database, the simulator shortcuts through a test endpoint.

That temptation is worth resisting. Once you have multiple code paths that can change state, you have multiple things to test, multiple places where bugs can hide, and a system that behaves differently depending on how a change was triggered. **The better approach is one code path for everything.**

## The pattern

Every state change in the system, regardless of source, publishes an MQTT message. The API subscribes to the broker and handles the update — writing to the database, triggering any downstream logic. Nothing writes state directly.

```
launderette/{site_id}/machine/{machine_id}/status
```

```json
{
  "machine_id": "wsh-01",
  "state": "Running",
  "substate": "Washing",
  "cycle_type": "Cotton_60",
  "drum_rpm": 0,
  "water_temp_c": 52,
  "elapsed_seconds": 420,
  "estimated_remaining_seconds": 1980
}
```

The hardware client publishes this. The WPF simulator publishes this. If the admin frontend ever needs to force a machine into a state, it publishes this — it does not call a state update endpoint directly.

<div class="callout"><p>The broker does not care who published the message. The subscriber does not care either. A message on the right topic with a valid payload gets processed. That's the entire contract.</p></div>

## Why it matters in practice

### Testing is honest

When the simulator publishes 1000 machine state changes and the subscriber processes them correctly, you have tested the real code path. Not a mock, not a shortcut. The same subscriber that handles real hardware handles the load test. If something breaks under volume, it would have broken with real machines too.

### The subscriber is the single source of truth logic

Validation, fault detection, Zabbix alerting — all of it lives in one place. When the subscriber receives a `Faulted` state, it handles it the same way regardless of whether a real machine sent it or a fault injection button in the simulator triggered it. You write that logic once.

### Adding new publishers is trivial

The WT32-ETH01 hardware client, the WPF simulator, a future mobile engineer app, a scheduled maintenance script — each one just needs to publish a valid payload to the right topic. The subscriber does not change. The database schema does not change. The monitoring does not change.

## The one tradeoff

Publishing directly from the browser or a simulator bypasses whatever validation your API might enforce before writing state. The broker forwards whatever arrives. **This is actually useful for a dev tool** — you want to inject bad states and edge cases — but be conscious that the simulator is not testing your validation logic, it is working around it by design. Keep that distinction clear.

For production hardware clients, input validation belongs in the subscriber before anything gets written to the database. Not in the publisher, not in the broker. In the subscriber, where you own the code.

## Setting up Mosquitto for this

One broker, two listeners. Port 1883 for raw MQTT (hardware clients, the API subscriber, the WPF simulator). Port 9001 for WebSockets if you ever need a browser-based client.

```
listener 1883
protocol mqtt

listener 9001
protocol websockets

allow_anonymous true
```

In production you would replace `allow_anonymous true` with proper ACL rules and TLS. For the homelab dev environment, getting the pattern working comes first.

The API subscriber connects on startup, subscribes to `launderette/#`, and processes everything that arrives. One connection, one handler, one code path. That is the entire point.
