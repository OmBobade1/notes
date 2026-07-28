## Level
Bandit28 → Bandit29

## Challenge
The password isn't in the current file content but was committed and later removed — it's visible in the commit history.

## Technique Used
Use `git log` and `git show`/`git diff` to inspect past commits.

## Commands
```bash
git clone ssh://bandit28-git@localhost:2220/home/bandit28-git/repo /tmp/repo28
cd /tmp/repo28 && git log -p
```
