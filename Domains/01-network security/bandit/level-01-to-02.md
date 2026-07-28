## Level
Bandit1 → Bandit2

## Challenge
The password is in a file called `-` in the home directory. A bare `cat -` reads from stdin instead of the file.

## Technique Used
Use a relative path or explicit stdin redirection so the shell doesn't treat `-` as an option flag.

## Commands
```bash
cat ./-
# or
cat < -
```
