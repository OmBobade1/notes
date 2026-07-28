## Level
Bandit32 → Bandit33

## Challenge
Logging in drops you into a shell that uppercases anything you type before executing it, breaking normal commands (including lowercase letters in shell syntax).

## Technique Used
Use shell metacharacters that survive the uppercasing (e.g. `$0` to spawn a fresh, non-mangled bash shell).

## Commands
```bash
$0
```
