## Level
Bandit6 → Bandit7

## Challenge
The file could be anywhere on the server, owned by a specific user and group, with a specific size.

## Technique Used
Use `find /` with `-user`, `-group`, `-size`, redirecting stderr to suppress permission-denied noise.

## Commands
```bash
find / -user bandit7 -group bandit6 -size 33c 2>/dev/null
```
