## Level
Bandit19 → Bandit20

## Challenge
A setuid binary owned by the next-level user can execute an arbitrary command as that user.

## Technique Used
Run the provided setuid binary with `cat` and the target password file as arguments.

## Commands
```bash
./bandit20-do cat /etc/bandit_pass/bandit20
```
