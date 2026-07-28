## Level
Bandit15 → Bandit16

## Challenge
The target service on port 30001 requires an encrypted (SSL) connection rather than a plain TCP one.

## Technique Used
Use `openssl s_client` to establish the encrypted connection, then send the password over it.

## Commands
```bash
echo <password> | openssl s_client -connect localhost:30001 -quiet
```
