## Level
Bandit11 → Bandit12

## Challenge
The password file's content has been ROT13-encoded (all letters rotated 13 places).

## Technique Used
Use `tr` to perform the ROT13 substitution manually.

## Commands
```bash
cat data.txt | tr 'A-Za-z' 'N-ZA-Mn-za-m'
```
