## Level
Bandit14 → Bandit15

## Challenge
The next password is obtained by sending the current password to a listening service on localhost:30000.

## Technique Used
Pipe/echo the password into `nc` connected to the target port.

## Commands
```bash
cat /etc/bandit_pass/bandit14 | nc localhost 30000
```
