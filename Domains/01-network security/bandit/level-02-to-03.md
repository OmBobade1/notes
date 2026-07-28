## Level
Bandit2 → Bandit3

## Challenge
The password file is named `spacefile name` (contains spaces), which breaks a naive `cat filename` call.

## Technique Used
Quote the filename (or escape the spaces) so it's passed as a single argument.

## Commands
```bash
cat "spacefile name"
# or
cat spacefile\ name
```
