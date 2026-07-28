## Level
Bandit25 → Bandit26

## Challenge
Login as this user drops you into a restricted program (e.g. `more`) instead of a normal shell.

## Technique Used
Trigger the pager on a large enough file so it doesn't fit on one screen, then use the pager's `!` (or `v`) escape to spawn a real shell.

## Commands
```bash
ssh -i bandit26.sshkey bandit26@localhost -p 2220
# inside the pager: press 'v' to open $EDITOR, then in vi: :set shell=/bin/bash  then  :shell
```
