## Level
Bandit20 → Bandit21

## Challenge
A setuid binary connects out to a port you specify and only sends the password if the connection comes from the same user.

## Technique Used
Start a local `nc` listener, then point the setuid binary's connect-back at that listener's port.

## Commands
```bash
nc -lvp 12345 &
./suconnect 12345
```
