# Auditing command-execution routes

A Next.js API route that spawns a shell is a **remote code execution
endpoint**. It must be treated as the most dangerous route in the app, not as
an internal utility.

## The recognition pattern

```
grep -rnE "execSync|exec\(|spawn\(|spawnSync|child_process|execFile" app/api/ src/app/api/
grep -rnE "shell:\s*true" app/api/ src/app/api/
```

Any hit is a candidate. For each one, answer three questions in order.

### 1. Is there real authentication?

Authentication is a **secret compared against an expected value**, checked
before the command runs. None of these count:

| Pattern | Why it is not auth |
|---|---|
| No check at all | Open RCE to anyone who can route to the app |
| A string-prefix allowlist on the submitted command | A prefix check constrains *what you may type*, not *who you are*. It also fails open when the prefix file does not exist |
| A hardcoded command list | Same problem — a list is not a credential |
| A header the client fully controls | That is a header, not a secret |
| IP allowlist / private-network assumption | Tailnets, VPNs, and reverse proxies all move the trust boundary. A "private" address is reachable by every device on that network |

**The prefix-allowlist trap.** A route that accepts a shell string and
gates it with something like `cmd.startsWith('python3 /path/to/tool.py ; ')`
is unauthenticated RCE with a costume. The prefix file may not even exist —
the prefix still satisfies the check, the interpreter errors to stderr, and
the shell's `;` continues into whatever follows. The allowlist protects
nothing once the prefix is known, and the prefix is not a secret.

The correct shape is: **secret first, then command shape.** Verify a shared
secret (compare with `crypto.timingSafeEqual`), *then* validate the command
against an allowlist — never instead of it.

### 2. What can the command reach?

A route that runs `bash -c` inherits the full privileges of the server
process. Inventory what that means:

- Which user does the server run as? A route running as the deploy user can
  read that user's credential files, SSH keys, and cloud credentials.
- Is the app reachable outside the machine? Dev servers bound to `0.0.0.0`,
  container ports published to the host, and Tailscale interfaces are all
  reachable off-box.
- Does the route expose stdout? Command output returned to the caller is a
  data-exfiltration channel even when the command itself is "read-only".

### 3. Can it be constrained without breaking the use case?

Most command-exec routes exist for one narrow job (run a maintenance script,
restart a service, inspect a log). Prefer:

- A **named operation** endpoint (`POST /api/system/restart-gateway`) that
  maps to a fixed argv — no caller-supplied string at all.
- A **fixed argv** with the dynamic part passed as a separate typed
  argument that is validated against an allowlist.
- Running as an unprivileged user with a restricted `PATH`.

## The tailnet case

A tailnet does not make a route private. Every device on the tailnet can
route to every other tailnet address. A command-exec route bound to a
tailnet IP is exposed to every machine the user has added — including
laptops, phones, and anything else that has joined.

## When you find one

Report it as **critical**, regardless of how internal it feels. State:

1. The route path and file:line.
2. What an unauthenticated caller can execute.
3. What the process identity gives them beyond that.
4. The concrete fix (named-operation endpoint, or secret-then-allowlist).

If the route is currently the only way to administer a remote machine,
say so — but that is a reason to add auth, not a reason to leave it open.

## Related

- SKILL.md §11 — timing-safe secret comparison.
- SKILL.md §12 — the subagent audit pattern.
- `references/security-audit-checklist.md` — the dispatch brief.
