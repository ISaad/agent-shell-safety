# Agent Shell Safety

Two failure classes that are easy to write wrong in agent-generated bash and
device/API automation, found and fixed in real production use (not theory).

## 1. Delayed parallel jobs: subshell scoping

**Symptom**: two commands meant to fire together after a shared delay — one
fires on time, the other fires immediately and/or in the wrong directory.

**Broken**:
```bash
sleep 30 && cd /some/dir && cmd1 &
cmd2 &
wait
```
Bash backgrounds the *whole* `sleep 30 && cd ... && cmd1` chain as one job.
`cmd2` is a separate, unchained background job — it runs immediately, in
whatever the original cwd was. Each command tested alone looks correct; the
bug only shows up when they're required to run in sync.

**Fixed**:
```bash
( sleep 30; cd /some/dir; cmd1 ) &
( sleep 30; cd /some/dir; cmd2 ) &
wait
```
Every backgrounded job gets its own subshell, with the delay and any `cd`
*inside* it.

**Verify**: run the real multi-job command once (not each command alone),
and print `date` before/after each job to confirm both start at the same
offset.

## 2. Restore-after-mutation: don't blindly restore every snapshot key

**Symptom**: code reads state, mutates it temporarily, restores from the
snapshot afterward — and the restore call itself fails or leaves the
resource in the wrong state, with a generic "invalid params" error that
looks like a permissions/lock issue.

**Broken**:
```python
restore = {"state": was_on}
for key in ("dimming", "temp", "sceneId", "r", "g", "b"):
    if key in before:
        restore[key] = before[key]
set_pilot(ip, restore)
```
If the target API rejects certain key *combinations* (e.g. `sceneId: 0`
alongside `temp`/`r,g,b` — "no scene" plus a direct color command), the call
fails generically. That generic error reads identically to a real
permission/lock error, so it's easy to burn time debugging the wrong
hypothesis while the resource stays broken on every call through that path,
not just the one you're testing.

**Fixed**:
```python
for key in ("dimming", "temp", "sceneId", "r", "g", "b"):
    if key == "sceneId" and before.get(key) == 0:
        continue
    if key in before:
        restore[key] = before[key]
```
Skip snapshot keys whose *value* means "not applicable" instead of restoring
every key that happened to be present.

**Verify**: after mutate+restore, re-fetch state and diff it against the
pre-call snapshot — don't just check for absence of an error. A generic
invalid-params error on a multi-key restore call is a cue to check for
conflicting keys before assuming a lock/permission cause.

## Checklist

- [ ] Backgrounded jobs sharing a delay: each is its own `( ... ) &` subshell
      with the delay/`cd` inside it. Tested as the real multi-job command,
      not each command alone.
- [ ] Mutate-then-restore code: re-fetch and diff state after restore. Any
      "invalid params" error on a multi-key write is checked for conflicting
      keys before assuming a lock/permission cause.

---

Found this useful? [Sponsor](https://github.com/sponsors/ISaad) — keeps
patterns like this documented as they're found.
