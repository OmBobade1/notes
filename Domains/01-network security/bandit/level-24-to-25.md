## Level
Bandit24 → Bandit25

## Challenge
A service on a port accepts the current password plus a 4-digit PIN in one line; the PIN must be brute-forced.

## Technique Used
Write a small script that iterates all 0000-9999 combinations and sends each to the service via `nc`.

## Commands
```bash
for i in $(seq -w 0 9999); do echo "<password> $i"; done > pins.txt
cat pins.txt | nc localhost <port>   # grep the output for the line containing 'Correct'
```
