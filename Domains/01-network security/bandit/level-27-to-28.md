## Level
Bandit27 → Bandit28

## Challenge
The password for the next level is stored inside a git repository you must clone.

## Technique Used
Clone the repo using the credentials for the current level and inspect its files.

## Commands
```bash
git clone ssh://bandit27-git@localhost:2220/home/bandit27-git/repo /tmp/repo27
cat /tmp/repo27/README
```
