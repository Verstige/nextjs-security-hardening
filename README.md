# Next.js Security Hardening: Headers, Bots, Limits

> The pre-deploy security pass for a Next.js app: bot protection, DevTools deterrence, rate limiting, security headers and audit logging, plus a subagent-driven audit that finds real issues.

![License: MIT](https://img.shields.io/badge/License-MIT-green.svg) ![Claude Code](https://img.shields.io/badge/Claude_Code-skill-blueviolet.svg) [![Available on Agensi](https://img.shields.io/badge/Agensi-%2418-ff5a5f.svg)](https://agensi.io/skills/next-js-security-hardening-headers-bots-limits)

## What it does

Harden a Next.js app: AI bot protection, DevTools deterrence, rate limiting, security headers, and an audit of what is actually exposed.

Comprehensive security layer for Next.js apps — bot protection, DevTools deterrence, rate limiting, audit logging, and a **pre-deployment security audit pattern** that catches 7+ critical classes of vulnerabilities in a single pass.

## The failure it prevents

Each reference file is one specific failure: the symptom, the actual cause, and the fix. The ones that come up most often:

- Auditing command-execution routes
- AI/ML Crawler Blocklist — robots.txt
- Mobile Pass A — Touch Targets, Safe-Area, 16px Font
- Pre-Deployment Security Audit — Grep + Checklist

## When to use it

- harden a Next.js app before launch
- add rate limiting and security headers
- run a pre-deployment security audit

## What's inside

```
plugins/
└── nextjs-security-hardening/
    ├── .claude-plugin/plugin.json
    └── skills/nextjs-security-hardening/
        ├── SKILL.md
        ├── references/            (4 files)
        │   └── command-execution-route-audit.md
        │   └── crawler-blocklist.md
        │   └── mobile-pass-a-css.md
        │   └── security-audit-checklist.md
```

## Install

Add this repo as a Claude Code marketplace, then install the skill:

```bash
claude plugin marketplace add Verstige/nextjs-security-hardening
claude plugin install nextjs-security-hardening@nextjs-security-hardening
```

Or copy the skill straight into your skills directory:

```bash
git clone https://github.com/Verstige/nextjs-security-hardening.git
cp -R nextjs-security-hardening/plugins/nextjs-security-hardening/skills/nextjs-security-hardening ~/.claude/skills/
```

## Usage

The skill activates on the work it describes. Ask for the outcome, and it
supplies the method:

```
> "harden a Next.js app before launch"
```

## Reference material

Loaded on demand — the skill only reads what the task needs.

| File | Covers |
| --- | --- |
| `command-execution-route-audit.md` | Auditing command-execution routes |
| `crawler-blocklist.md` | AI/ML Crawler Blocklist — robots.txt |
| `mobile-pass-a-css.md` | Mobile Pass A — Touch Targets, Safe-Area, 16px Font |
| `security-audit-checklist.md` | Pre-Deployment Security Audit — Grep + Checklist |

## License

MIT. See [LICENSE](LICENSE).

---

**Also available as a paid skill on Agensi** — $18. Install it with one command, no clone required:

[agensi.io/skills/next-js-security-hardening-headers-bots-limits](https://agensi.io/skills/next-js-security-hardening-headers-bots-limits)

Part of the [Verstige](https://github.com/Verstige) skills portfolio. More at [github.com/Verstige](https://github.com/Verstige).
