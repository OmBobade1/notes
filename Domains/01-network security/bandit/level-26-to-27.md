## Level
Bandit26 → Bandit27

## Challenge
Similar restricted-shell setup; escalate through an editor launched from within the pager to get a real shell, then explore as the next user.

## Technique Used
From inside the spawned editor, invoke a shell command to obtain an interactive bash session.

## Commands
```bash
:set shell=/bin/bash
:shell
# once in a shell:
ls -la
find / -user bandit27 2>/dev/null
```
