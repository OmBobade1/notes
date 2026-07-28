## Level
Bandit16 → Bandit17

## Challenge
A range of ports (31000-32000) is open; only one accepts an SSL connection and returns a private key for the next level.

## Technique Used
Scan the port range with `nmap`, test each open port with `openssl s_client`, then save the returned private key.

## Commands
```bash
nmap -p 31000-32000 localhost
openssl s_client -connect localhost:<port> -quiet   # try each open port
# paste the returned RSA private key into a local file, then:
chmod 600 key.private
ssh -i key.private bandit17@localhost -p 2220
```
