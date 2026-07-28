## Level
Bandit17 → Bandit18

## Challenge
Two files (`passwords.old` and `passwords.new`) differ by exactly one line, which contains the new password.

## Technique Used
Use `diff` to compare the two files and isolate the changed line.

## Commands
```bash
diff passwords.old passwords.new
```
