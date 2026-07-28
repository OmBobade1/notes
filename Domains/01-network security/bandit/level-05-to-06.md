## Level
Bandit5 → Bandit6

## Challenge
The password is in a file somewhere under a directory tree, matching specific size (1033 bytes), and being human-readable/not executable.

## Technique Used
Use `find` with `-size` and permission filters.

## Commands
```bash
find . -size 1033c -not -type d
```
