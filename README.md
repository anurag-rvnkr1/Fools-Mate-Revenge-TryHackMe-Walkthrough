# ♟ Fool's Mate, Revenge — TryHackMe CTF Walkthrough

> **Room:** [Fool's Mate, Revenge](https://tryhackme.com/room/foolsm8v2)  
> **Category:** Web Security  
> **Difficulty:** Advanced  
> **Primary Technique:** Prototype Pollution  
> **Target:** Node.js chess application on port 3000

A premium, portfolio-ready technical walkthrough documenting the complete attack path against the hardened sequel to the original Fool's Mate challenge.

The key lesson is that **server-side authorization can still fail when attacker-controlled objects are merged unsafely**. The application correctly moved the security gate into the backend, but a vulnerable deep-merge routine allowed the JavaScript object prototype to be manipulated, causing the reward gate to resolve as unlocked.

## Attack Chain

`Recon → API Discovery → Authorization Gate Analysis → JavaScript Review → Prototype Pollution → State Manipulation → Checkmate → Reward`

## Executive Findings

| Finding | Severity | Impact |
|---|---:|---|
| Prototype Pollution in deep-merge logic | Critical | Enables modification of inherited application state |
| Unsafe handling of `constructor.prototype` | Critical | Permits prototype-level property injection |
| Reward gate trusts inherited state | High | Converts object pollution into authorization bypass |
| Debug/stack-trace leakage during exploitation | Medium | Exposes server implementation details |

## Repository Layout

```text
.
├── Documentation/
│   ├── Documentation.md
│   ├── Documentation.docx
│   └── README.md
├── Resources/
│   └── notes.md
├── Screenshots/
│   ├── 01-room-overview.png
│   ├── 02-endgame-position.png
│   ├── 03-preferences-surface.png
│   ├── 04-prototype-pollution-exploitation.png
│   └── 05-checkmate-reward.png
├── docs/
│   ├── index.md
│   └── assets/
│       ├── 01-room-overview.png
│       ├── 02-endgame-position.png
│       ├── 03-preferences-surface.png
│       ├── 04-prototype-pollution-exploitation.png
│       ├── 05-checkmate-reward.png
│       └── css/custom.scss
├── _config.yml
├── LICENSE
└── README.md
```

## Public Flag Policy

The final challenge flag is intentionally redacted in the public portfolio version:

`THM{REDACTED_FOR_PUBLICATION}`

The actual flag is present only in the author's private notes/source evidence, not in the public Markdown or DOCX.

## Documentation

- [Full Technical Walkthrough](Documentation/Documentation.md)
- [Word Version](Documentation/Documentation.docx)
- [GitHub Pages Version](docs/index.md)
- [Technical Notes](Resources/notes.md)

## Responsible Disclosure / Lab Safety

This write-up documents an authorized TryHackMe laboratory environment. Do not reproduce these techniques against systems without explicit authorization.

## Author

**Anurag Revankar**  
[GitHub](https://github.com/anurag-rvnkr1)
